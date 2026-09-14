# Stop Re-Downloading the NVD: A Shared Local Vulnerability Database for Jenkins

*Setting up OWASP Dependency-Check 12.1.7 with a shared H2 database on a Jenkins controller*

---

## The problem

OWASP Dependency-Check is a great tool right up until you run it on a busy Jenkins controller.

Out of the box, every single job that invokes Dependency-Check wants its own copy of the National Vulnerability Database. Each job downloads it. Each job caches it in its own workspace. And because the NVD API is rate-limited, those downloads queue up behind each other, fail intermittently, and turn a two-minute scan into a twenty-minute one. Run three jobs in parallel and you'll start seeing timeouts and half-populated databases.

The fix is conceptually simple: **download the NVD data once, centrally, and have every job read from that shared copy.**

That's what this guide sets up. One H2 database on the Jenkins controller, refreshed by a single cron job at 2am, with every pipeline pointed at it in read-only mode via `--noupdate`. Jobs stop touching the internet entirely. Scans get fast and deterministic.

### What you end up with

```
┌─────────────────────────────────────────┐
│          Jenkins Controller VM          │
│                                         │
│  ┌─────────────────┐                    │
│  │  H2 TCP Server  │ ← systemd service  │
│  │  port 9092      │ ← auto-starts      │
│  │  /opt/dc-h2-data│ ← DB files         │
│  └────────┬────────┘                    │
│           │                             │
│  ┌────────┴────────┐                    │
│  │   Cron Job      │ ← 2am daily        │
│  │   --updateonly  │ ← pulls from NVD   │
│  └─────────────────┘                    │
│                                         │
│  ┌─────────────────┐                    │
│  │  Jenkins Jobs   │ ← --noupdate       │
│  │  (pipeline)     │ ← reads local DB   │
│  └─────────────────┘                    │
└─────────────────────────────────────────┘
         ↑
    NVD API (once daily only)
```

### Environment

This was built and tested on:

| | |
|---|---|
| **OS** | Ubuntu (Jenkins controller VM) |
| **Dependency-Check** | 12.1.7 |
| **Java** | OpenJDK 21 |
| **Database** | H2 2.3.232, TCP server mode |

Other versions will mostly work, but the H2 and dependency-check-core jar filenames will differ — adjust as you go.

### Before you start

- Jenkins installed and running
- The OWASP Dependency-Check plugin installed
- Dependency-Check registered as a Tool under **Manage Jenkins → Tools** (write down the exact name you gave it — you'll need it later, and typos here are a classic time sink)
- An NVD API key from [nvd.nist.gov/developers/request-an-api-key](https://nvd.nist.gov/developers/request-an-api-key). Without one you're throttled to a trickle; with one the initial load is merely slow rather than glacial.

---

## Step 1: Find your installation paths

Jenkins buries tool installations under a long auto-generated directory name, and you'll reference these paths in roughly every command that follows. Get them once, up front:

```bash
# The main binary
find /var/lib/jenkins/tools -iname "dependency-check.sh"

# The lib directory (contains the h2 and core jars)
find /var/lib/jenkins/tools -iname "h2-*.jar"
find /var/lib/jenkins/tools -iname "dependency-check-core*.jar"
```

Throughout the rest of this guide I'll refer to the lib directory as `<LIBDIR>`. On my box it was:

```
/var/lib/jenkins/tools/org.jenkinsci.plugins.DependencyCheck.tools.DependencyCheckInstallation/DependecyCheck/lib
```

Note the misspelling in `DependecyCheck` — that's the tool name I typed into Jenkins, and Jenkins faithfully used it for the directory. Yours will reflect whatever you named it. Substitute your own path everywhere you see `<LIBDIR>`.

---

## Step 2: Create a home for the database

The H2 data files need to live somewhere outside any job workspace, owned by the user Jenkins runs as:

```bash
sudo mkdir -p /opt/dc-h2-data
sudo chown jenkins:jenkins /opt/dc-h2-data
```

---

## Step 3: Run H2 as a systemd service

Dependency-Check can use an embedded H2 database, but embedded mode means one process at a time holds a lock on the file. That's exactly what we're trying to avoid. Running H2 in **TCP server mode** lets the nightly updater and all your concurrent jobs talk to the same database at once.

Making it a systemd unit means it comes back after a reboot, which — spoiler — is the failure mode that bites people three weeks later when the VM gets patched.

```bash
sudo tee /etc/systemd/system/h2-dependencycheck.service > /dev/null <<EOF
[Unit]
Description=H2 Database Server for OWASP Dependency-Check
After=network.target

[Service]
User=jenkins
ExecStart=/usr/bin/java -cp "<LIBDIR>/*" org.h2.tools.Server -tcp -tcpAllowOthers -tcpPort 9092 -baseDir /opt/dc-h2-data -ifNotExists
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

**The classpath detail that matters.** It's tempting to put just `h2-2.3.232.jar` on the classpath — it's an H2 server, after all. Don't. Dependency-Check's schema uses stored procedures that call into Guava, and if Guava isn't on the server's classpath you'll get a cryptic `com/google/common/base/Strings` error partway through your first data load, after you've already waited forty minutes.

Pointing at `"<LIBDIR>/*"` loads every jar in the directory and sidesteps the whole problem. Keep the quotes: you want Java to expand that wildcard, not your shell.

Enable and start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now h2-dependencycheck
sudo systemctl status h2-dependencycheck
```

You want `Active: active (running)`. Confirm it's actually listening:

```bash
sudo ss -ltnp | grep 9092
```

---

## Step 4: Extract the schema script

Dependency-Check ships its schema as `initialize.sql` inside the core jar. Pulling it from the jar rather than downloading it from GitHub guarantees the schema matches the version you actually have installed — schema versions do change between releases, and a mismatch produces confusing errors at scan time rather than at setup time.

```bash
cd /tmp
unzip -p <LIBDIR>/dependency-check-core-12.1.7.jar data/initialize.sql > initialize.sql
head -20 /tmp/initialize.sql
```

If `head` shows you SQL, you're good.

---

## Step 5: Initialise the schema

```bash
sudo -u jenkins java -cp <LIBDIR>/h2-2.3.232.jar org.h2.tools.RunScript \
  -url "jdbc:h2:tcp://localhost:9092/dependencycheck" \
  -user sa \
  -password "ChangeMe123!" \
  -script /tmp/initialize.sql
```

Two things worth knowing here:

**Use a real password.** H2 2.x refuses blank passwords over TCP connections. An empty string will get you `Wrong user name or password` and no useful hint as to why. Obviously pick something better than `ChangeMe123!` for a real deployment.

**Keep the script somewhere jenkins can read.** `/tmp` works. Running it out of your own home directory (`/home/yourname/...`) will throw an `AccessDeniedException`, because `sudo -u jenkins` means the jenkins user needs read access and it almost certainly doesn't have it.

Success here looks like *nothing at all* — no output, no exceptions. Quiet is good.

---

## Step 6: Put the connection details in a properties file

You can pass connection settings as command-line flags, but the Jenkins plugin mangles quoting in `additionalArguments` in ways that are incredibly painful to debug. A properties file avoids that entirely, and it keeps your database password out of the Jenkinsfile and therefore out of source control.

```bash
sudo tee /etc/dependency-check.properties > /dev/null <<EOF
data.connection_string=jdbc:h2:tcp://localhost:9092/dependencycheck
data.driver_name=
data.driver_path=
data.user=sa
data.password=ChangeMe123!
EOF
```

The two blank driver entries are intentional — they override any defaults so H2's bundled driver gets used.

Lock it down:

```bash
sudo chmod 600 /etc/dependency-check.properties
sudo chown jenkins:jenkins /etc/dependency-check.properties
```

---

## Step 7: The first full data load

This is the long one. It pulls the entire NVD dataset into your local database:

```bash
sudo -u jenkins <LIBDIR>/../bin/dependency-check.sh \
  --updateonly \
  --nvdApiKey YOUR_NVD_API_KEY \
  --propertyfile /etc/dependency-check.properties
```

Expect anywhere from a few minutes to over an hour depending on how the NVD API is feeling. You'll see CVE and CPE records scrolling past — that's the tool working, not spinning. **Don't interrupt it.** A partial load leaves you with a database that looks fine and reports incomplete results, which is worse than no database at all.

Go make coffee. This is a once-per-lifetime-of-the-VM operation.

---

## Step 8: Automate the daily refresh

New CVEs land constantly, so the database needs a nightly top-up. Unlike the initial load, incremental updates are quick.

Create the log file first, owned by jenkins:

```bash
sudo touch /var/log/jenkins/dc-update.log
sudo chown jenkins:jenkins /var/log/jenkins/dc-update.log
```

Then edit the jenkins user's crontab:

```bash
sudo crontab -u jenkins -e
```

Add:

```
0 2 * * * /path/to/dependency-check.sh --updateonly --nvdApiKey YOUR_NVD_API_KEY --propertyfile /etc/dependency-check.properties >> /var/log/jenkins/dc-update.log 2>&1
```

Obviously you'll need to replace `/path/to/dependency-check.sh` with the real binary path from Step 1. Confirm it saved:

```bash
sudo crontab -u jenkins -l
```

> **A note on the API key:** putting it directly in the crontab means it's readable by anyone who can read that crontab, and it'll show up in `ps` output while the job runs. If that matters in your environment, wrap the command in a small root-owned script with `chmod 700`, or export the key from a sourced env file instead.

It's worth checking `/var/log/jenkins/dc-update.log` a few mornings after setup. A silently failing update job is the kind of thing you don't notice until an audit asks why your scans missed a six-month-old CVE.

---

## Step 9: Point your pipelines at it

Now the payoff. Every job uses `--noupdate` (don't touch NVD) and `--propertyfile` (read from the shared database):

```groovy
pipeline {
    agent any

    stages {
        stage('Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '''
                    --noupdate
                    --scan .
                    --format ALL
                    --out dependency-check-report
                    --propertyfile /etc/dependency-check.properties
                ''', odcInstallation: 'DependecyCheck'
            }
        }

        stage('Publish Report') {
            steps {
                dependencyCheckPublisher pattern: 'dependency-check-report/dependency-check-report.xml'
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'dependency-check-report/**', allowEmptyArchive: true
        }
    }
}
```

`odcInstallation` must match the tool name from **Manage Jenkins → Tools** exactly — including any typo you made when you created it. Mine is `DependecyCheck`, missing an `n`. Yours is whatever yours is.

---

## Step 10: Reboot before you trust it

Everything can work perfectly and still fall over the first time the VM restarts, because `systemctl start` and `systemctl enable` are different things. Test it now rather than discovering it during a maintenance window:

```bash
sudo reboot
```

When it comes back up, walk through the checks:

**Did H2 auto-start?**
```bash
sudo systemctl status h2-dependencycheck   # want: Active: active (running)
sudo ss -ltnp | grep 9092
```

**Is the schema intact?**
```bash
sudo -u jenkins java -cp "<LIBDIR>/*" \
  org.h2.tools.Shell \
  -url "jdbc:h2:tcp://localhost:9092/dependencycheck" \
  -user sa \
  -password "ChangeMe123!" \
  -sql "SELECT value FROM properties WHERE id='version';"
```

You should get back a schema version number — something like `5.4`.

**Then run an actual Jenkins job** and confirm it completes with no database errors and a populated report.

---

## Troubleshooting

The errors I hit, and what they actually meant — and believe me, some of them were insanely dumb:

| Error | Cause | Fix |
|---|---|---|
| `No suitable driver found for jdbc:postgreql://...` | Typo in the connection string | Look closely — `postgreql` should be `postgresql` |
| `Address already in use` on port 9092 | An earlier H2 process still running | `sudo lsof -i :9092`, then `sudo kill -9 <PID>` |
| `Table "PROPERTIES" not found` | Schema never initialised | Re-run Step 5 |
| `Wrong user name or password` | H2 2.x rejects blank passwords over TCP | Set a real password |
| `com/google/common/base/Strings` in a stored procedure | Guava missing from the H2 classpath | Use `"<LIBDIR>/*"` in `ExecStart`, not just the h2 jar |
| `AccessDeniedException` on initialize.sql | jenkins can't read the file | Move it to `/tmp`, `chmod 644` |
| `Unable to connect to the database` in Jenkins | Quoting mangled in `additionalArguments` | Use `--propertyfile` rather than inline connection args |
| `Unable to connect to the database` after a reboot | Service was started but never enabled | `sudo systemctl enable h2-dependencycheck` |

---

## Was it worth it?

Yes — the NVD download disappears from every build, scans become consistent, and parallel jobs stop fighting each other over the database. The trade-off is one more service to keep an eye on and a nightly job that needs monitoring.

If you're scanning from more than one build node, the same H2 server will serve them too, since it's already listening on TCP with `-tcpAllowOthers`. Just restrict access to port 9092 at the firewall so it's only reachable from your build network — `tcpAllowOthers` means exactly what it says.

---

*Written up for dependency-check 12.1.7 on Ubuntu with OpenJDK 21. Paths and jar versions will differ on other setups — adjust accordingly.*

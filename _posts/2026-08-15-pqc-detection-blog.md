# Is Your TLS Stack Quantum-Ready? Why PQC Detection Is Harder Than It Looks

*A look at the post-quantum transition, the harvest-now-decrypt-later threat model, and what it actually takes to tell whether an endpoint has migrated.*

---

## The problem nobody can postpone

Almost every secure connection on the internet today rests on the same small set of assumptions: that factoring large integers is hard, that discrete logarithms are hard, that elliptic-curve discrete logarithms are hard. RSA, Diffie-Hellman and ECC all inherit their security from these problems.

Shor's algorithm dissolves all three. Not slowly — efficiently. The only thing standing between us and that outcome is engineering: nobody has yet built a quantum computer large and stable enough to run it at the scale required. A machine that could is usually called a **cryptographically relevant quantum computer (CRQC)**.

No publicly acknowledged CRQC exists. That is the comforting half of the sentence. The uncomfortable half is that it doesn't matter as much as people assume.

## Harvest now, decrypt later

The threat model that makes this urgent is **harvest now, decrypt later (HNDL)**. An adversary with the storage capacity and the patience doesn't need a quantum computer today. They need a hard drive.

Encrypted traffic captured in 2026 and archived can sit untouched until a CRQC arrives, at which point it is retroactively readable. Every session key negotiated under classical public-key cryptography becomes recoverable, and with it everything that session protected.

The practical consequence is a simple piece of arithmetic:

> **If the confidentiality lifetime of your data exceeds the time until a CRQC exists, that data is already compromised — you just don't know it yet.**

For a session cookie, this is irrelevant. For medical records, financial histories, legal discovery, source code, trade secrets, diplomatic cables, or anything with a twenty-year secrecy requirement, it is the whole problem. Migration cannot wait for proof that quantum computers work, because by then the harvested data is already lost.

## What NIST already standardised

NIST ran a multi-round public competition starting in 2016 and began publishing finalised standards in 2024. It's important to be precise about the status of each, because a lot of writing on this topic blurs them together:

| FIPS | NIST name | Former name | Type | Status |
|---|---|---|---|---|
| 203 | **ML-KEM** | CRYSTALS-Kyber | Lattice-based KEM | Final (Aug 2024) |
| 204 | **ML-DSA** | CRYSTALS-Dilithium | Lattice-based signature | Final (Aug 2024) |
| 205 | **SLH-DSA** | SPHINCS+ | Hash-based signature | Final (Aug 2024) |
| 206 | **FN-DSA** | FALCON | NTRU-lattice signature | Draft — final expected late 2026 / early 2027 |

Three finalised, one still in draft. FN-DSA took longer largely because its signing routine depends on floating-point Gaussian sampling that is genuinely difficult to implement safely and in constant time.

The split of responsibilities is worth internalising:

- **ML-KEM** replaces the key exchange. This is the part that matters for HNDL, because key exchange is what protects confidentiality of the session.
- **ML-DSA, SLH-DSA and FN-DSA** replace signatures. These matter for authentication — proving a server is who it claims to be. Signatures resist harvesting attacks by nature: forging a signature after the fact doesn't retroactively break a connection that already happened.

That asymmetry explains real-world deployment patterns. Cloudflare, Google and others rolled out hybrid ML-KEM key exchange years before anyone seriously deployed PQC certificates. Confidentiality was the bleeding wound; authentication can wait for the CA ecosystem to catch up.

## Why "just check if it's quantum-safe" is not a one-liner

Here's where the tooling gap opens up. If you ask a security team which of their endpoints are PQC-enabled, most cannot answer. Not because they're negligent, but because the information isn't in one place.

PQC can appear at several different layers of a TLS/HTTP deployment, independently of each other:

**1. The key exchange.** During a TLS 1.3 handshake, client and server negotiate a *named group* for key agreement. PQC deployments advertise post-quantum or hybrid groups here. A server can be running ML-KEM key exchange while presenting an entirely classical RSA certificate — extremely common, and the single most valuable thing to detect, since it's the HNDL mitigation.

**2. The certificate.** X.509 certificates identify their algorithms by **OID** (Object Identifier) in several places: the signature algorithm, the subject public key info, and occasionally in extensions. A PQC certificate carries PQC OIDs. A hybrid or dual-certificate deployment may present both classical and post-quantum chains.

**3. The cipher suite.** Some TLS libraries and HSM vendors surface PQC algorithm names inside custom cipher suite identifiers rather than through standard negotiation.

**4. HTTP response metadata.** CDN and edge providers sometimes advertise PQC support through response headers or `Alt-Svc` entries. This one is a bit of an outlier — but it's the only signal available when TLS terminates at an edge node you can't reach directly.

Check only one of these and you will produce confidently wrong answers. A tool that looks solely at certificates will report Cloudflare-fronted properties as "not quantum-ready" despite them running ML-KEM key exchange. A tool that looks solely at the handshake will miss a PKI that has deployed PQC certificates but not yet PQC key agreement.

### The identifier fragmentation problem

There's a second difficulty that surprises people. **The same algorithm has multiple identifiers in the wild, and all of them are still in circulation.**

PQC standardisation spanned nearly a decade. During that time, implementations shipped against draft specifications, research libraries assigned themselves experimental OIDs, and named group IDs were allocated, deprecated and reallocated. What's deployed today is a mix of:

- Final NIST-assigned OIDs
- IETF draft-era OIDs
- Experimental round-3 OIDs from research implementations
- Pre-standardisation hybrid group IDs from early vendor deployments
- Finalised IETF hybrid and standalone group IDs

An ML-KEM-768 deployment might identify itself in any of several ways depending on when it was configured and what library built it. Detection tooling that only knows the final identifiers will produce false negatives on a significant slice of real infrastructure — often the very infrastructure that adopted PQC earliest.

Handling this means maintaining a mapping across every namespace, for every algorithm and variant, and keeping it current as IANA and IETF assignments evolve. It's unglamorous and thankless work, and, at the same time, it's most of what separates detection that works from detection that looks like it works.

## What good detection has to get right

Given all of the above, a few properties matter more than they might appear to at first.

**It should be agentless.** Detection can work entirely from externally observable protocol behaviour — nothing installed on the target, no credentials, no configuration read. That constraint isn't a limitation, it's the point: if all you need is a TCP connection, then third-party endpoints, vendor APIs and CDN properties are in scope too, not just infrastructure you control.

**It shouldn't phone home.** Probes should connect only to the target. For anyone assessing internal network inventory, this is non-negotiable: a list of your internal hostnames annotated with their cryptographic weaknesses is exactly the kind of thing that shouldn't leave the building through a third-party scanning API.

**It should query every surface, every time.** Handshake negotiation, certificate inspection, cipher suite analysis and HTTP metadata, aggregated before any verdict is formed. Partial deployments — which is nearly all of them right now — then get represented accurately instead of being rounded to yes or no.

**It should emit structured output.** Findings that land in a screenshot die there. Machine-readable results can feed SIEM ingestion, pipeline gating and compliance reporting, which is where they're actually useful.

### Turning findings into a posture

Raw detections are not an answer. What a security team needs is triage, and that means collapsing findings into a small number of tiers based on how many of the NIST algorithm groups are in evidence across all probed ports and surfaces:

| Groups detected | Posture | What it means |
|---|---|---|
| 4 | **Fully quantum-ready** | All four NIST algorithm groups actively deployed |
| 2–3 | **Partially quantum-ready** | Migration is genuinely underway |
| 1 | **Minimal PQC adoption** | Early-stage; a migration plan is needed |
| 0 | **Not quantum-ready** | Exposed to harvest-now-decrypt-later |

Four tiers rather than a numeric score, and deliberately so. A 0–100 score invites debate about weighting and gives teams something to optimise instead of something to fix. A handful of labels gives you a sort order: everything in the bottom tier first.

### Where this pays off

Four workflows come up again and again:

- **Baseline assessment.** Survey the full TLS estate — public and internal — before a migration programme starts, so effort goes where the exposure actually is rather than where it's assumed to be.
- **CI/CD gating.** Treat PQC regression as a deployment-blocking defect. If a configuration change silently drops post-quantum key exchange, the pipeline should catch it, not a customer.
- **PKI auditing.** Teams running dual-certificate strategies need to verify that both chains are presented correctly and that PQC identifiers land in the fields they're supposed to.
- **Supply-chain review.** Vendor assessments can cite objective evidence of whether a SaaS provider, payment processor or API partner has actually started migrating, instead of accepting a questionnaire answer.

## Closing Thoughts

The cryptographic transition now underway is the largest since the introduction of public-key cryptography, and it will take years. The first step in any migration is inventory — you cannot migrate what you cannot see, and you cannot prioritise what you have not measured.

Most organisations are still at that first step. Detection tooling isn't the interesting part of the post-quantum problem, but it's the part that has to come first.

---

### References

1. NIST. *FIPS 203: Module-Lattice-Based Key-Encapsulation Mechanism Standard.* August 2024.
2. NIST. *FIPS 204: Module-Lattice-Based Digital Signature Standard.* August 2024.
3. NIST. *FIPS 205: Stateless Hash-Based Digital Signature Standard.* August 2024.
4. NIST. *FIPS 206: FN-DSA.* Draft; final publication anticipated late 2026 / early 2027.
5. NSA. *Commercial National Security Algorithm Suite 2.0.* CNSSP No. 15, 2022.
6. CISA. *Post-Quantum Cryptography Initiative.*
7. Bernstein, D. and Lange, T. *Post-quantum cryptography.* Nature 549, 188–194 (2017).

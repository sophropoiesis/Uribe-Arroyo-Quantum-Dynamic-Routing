# Uribe Arroyo Quantum Dynamic Routing (UAQDR)

## A Proposed Feedback-Controlled, Multi-Source Quantum Entropy Architecture

**Esteban Uribe Arroyo**  
**Technical research draft • Version 0.1 • 13 November 2026**  
**Status:** Proposal; no implemented prototype, certification, independent evaluation, or verified security advantage is claimed in this draft.

### Abstract

Uribe Arroyo Quantum Dynamic Routing (UAQDR) is a proposed entropy-acquisition and conditioning architecture that combines measurements from multiple quantum random-number generators (QRNGs) and employs quantum-derived control material to select the order or route of subsequent measurements. Unlike a static pipeline, the routing schedule changes between acquisition epochs according to private, cryptographically conditioned measurements collected in previous epochs. The proposed design incorporates per-source health monitoring, source provenance, bounded scheduling, cryptographic conditioning, separation between output and control uses, and conservative failure handling. The central research question is whether this feedback-controlled routing improves measurable robustness, fault isolation, or resistance to specific observables relative to matched multi-source static architectures. Dynamic scheduling is **not** itself a source of new entropy, a proof of unpredictability, or a proven side-channel countermeasure. The paper defines a conceptual architecture and falsifiable evaluation plan, not a validated cryptographic primitive.

**Keywords:** quantum random-number generation; QRNG; entropy pooling; random bit generation; dynamic routing; min-entropy; entropy-source health testing; cryptographic conditioning; physical randomness.

---

## 1. Introduction and contributions

Reliable generation of unpredictable key material depends on entropy sources and on the integrity of their acquisition, health tests, and conditioning. Physical QRNGs can exploit quantum processes, but practical systems contain non-ideal detectors, classical noise, firmware, shared clocks, interfaces, and environmental influences. Multiple sources may improve availability and fault tolerance *when their dependence and failures are characterized*.

UAQDR proposes a *closed-loop acquisition controller* in which fresh QRNG-originated control material chooses a later read schedule, followed by pool conditioning and domain-separated derivation of externally delivered output. The initial contributions asserted by this draft are a specification and a test plan, **not** a claim of invention priority, novelty established against all prior art, or improved security.

**Research questions**

- **RQ1 — Conditional entropy:** Under what assumptions, if any, does the adaptive schedule preserve an entropy lower bound compared with a fixed order?
- **RQ2 — Degraded-source resilience:** Does multiple-source ingestion improve availability under isolated QRNG faults without disguising common-mode failure?
- **RQ3 — Leakage:** Does schedule variation change timing, bus, electromagnetic, or power-leakage distinguishability, and can it ever make leakage worse?
- **RQ4 — Costs:** What are the added latency, energy, hardware, synchronization, and testability costs?

## 2. Related work and limits of novelty

The concepts of entropy accumulation, multiple pools, source identifiers, reseeding, and cryptographic conditioning predate UAQDR. The Fortuna accumulator is an important design precedent. Research on QRNG extraction emphasizes that raw quantum measurements require statistical modeling and extraction. Existing hardware random-number-generator research has also examined random routing across multiple entropy elements. UAQDR must be compared with these approaches before any novelty or patentability assertion. The proposed differentiator is the **particular feedback-controlled routing specification coupled to independently monitored QRNG sources**; whether this is new or beneficial remains open.

## 3. Architecture and trust boundaries

### 3.1 Components

1. **QRNG sources** `Q_1, …, Q_n`: physically separated where feasible, each with known sampling characteristics and per-source raw-data health tests.
2. **Collection adapters:** tag samples with source ID, epoch, sequence number, read success, and health state; control interfaces and timeouts are trust boundaries.
3. **Routing controller:** uses *prior epoch* confidential conditioned control material to choose a bounded schedule. The controller must never use a public output stream as a secret routing seed.
4. **Accumulation and conditioning:** combine accepted samples in an unambiguous canonical encoding; condition using a vetted cryptographic construction, not an improvised reversible mixer.
5. **Output interface:** release only conditioned output; limit output to supported entropy/security strength; expose structured health status independently from raw secret measurements.

### 3.2 Conceptual data flow

```text
  Quantum-source bank:   Q1       Q2       Q3       ... Qn
                          \        |        /             /
                           \ per-source raw health tests /
                            \      |       /            /
                             Collection adapters & provenance
                                          |
            prior conditioned control --> Routing scheduler
                                          |
                                Bounded acquisition epoch
                                          |
                           Canonical framing and entropy pool
                                          |
                            Cryptographic conditioning / KDF
                                   /                 \
                        output-domain key       next-route key
                              |                       |
                      controlled output       next epoch routing
```

*The diagram describes the design; it is not evidence of security.*

### 3.3 Conservative schedule (reference candidate)

The reference candidate samples *each healthy required source at least once* per epoch and changes only the order, not the minimum inclusion quota. This isolates routing variation from entropy omission during initial experiments. Optional additional reads are a distinct experimental mode and require separate analysis.

For epoch `t`, let `A_t` denote the eligible, health-passing source set. Derive an internal seed `r_t` from conditioned, non-public control material of a previous accepted epoch; bootstrap with independently collected startup entropy after initialization and health checks. Let

$$
\pi_t = \mathrm{UnbiasedShuffle}(A_t; \operatorname{CSPRNG}(r_t,\text{``route''}\|t))
$$

where `UnbiasedShuffle` uses rejection sampling or an equivalent bias-free bounded-range sampler rather than naïve modulo reduction. Collect `X_{i,t}` with an enforced quota from each selected source in the scheduled order. Include stable source identifiers and explicit lengths in a canonical frame `F_t`, so that different input sequences cannot encode to the same ambiguous byte string.

One candidate *computational conditioner* can be modeled as

$$
 K_t = \mathrm{HKDF\mbox{-}Extract}(\mathrm{salt}_t,F_t), \qquad
 Y_t = \mathrm{HKDF\mbox{-}Expand}(K_t,\text{``UAQDR/output''}\|t,L),
$$

$$
 r_{t+1} = \mathrm{HKDF\mbox{-}Expand}(K_t,\text{``UAQDR/route''}\|t,\ell_r).
$$

These equations define a **design candidate only**, not an approved NIST SP 800-90B conditioner or a formal security reduction. `salt_t` must be explicitly specified and managed; length `L`, derived key lengths, extraction claims, and reseeding frequency must be selected based on conservatively justified entropy. Standard HKDF analysis is computational and does not establish unconditional extraction of biased or adversarially correlated QRNG data. The same input material may be used for domain-separated derivations, but **its entropy must not be counted twice**. A standards-aligned implementation should select and validate conditioning/RBG construction choices against the full requirements of SP 800-90B and SP 800-90C.

**Avoiding a circular bootstrap:** A fresh quantum output cannot choose its own acquisition path before it exists. The first scheduling seed must come from a fixed, documented bootstrap collection process; subsequent epochs use accepted prior material. Loss of internal state must trigger a reinitialization process.

## 4. Entropy model and assumptions

For raw random variable `X`, the ordinary min-entropy is

$$
H_\infty(X)=-\log_2 \max_x \Pr[X=x].
$$

For adversarial side information `E` and adaptive schedule/history `H_t`, an operational entropy claim must conservatively address $H_\infty(X_{i,t}\mid E,H_t)$ under an explicit definition of conditional min-entropy and a source model. It is incorrect to sum per-device entropy estimates merely because devices have different serial numbers: dependence, shared power/clock, environment, malicious firmware, and schedule selection can invalidate an independence assumption. Aggregated entropy cannot exceed the entropy supported by the underlying measurements; conditioning and routing do not manufacture unknown physical information.

The paper presently **does not assign a bit-level entropy guarantee**. A later empirical section must specify source distributions, bounds and confidence levels, operational conditions, adversary capabilities, correlation tests, and the permitted output rate. Passing a statistical test suite alone cannot demonstrate cryptographic unpredictability.

## 5. Threat model

**In scope for evaluation:** a failed or biased source; partial compromise of one collection adapter; correlated failures; adversary observation of exposed output and externally measurable timing or power; attempted read-schedule inference; clock drift, timeout, and data replay; denial-of-service through faulty devices.

**Out of scope / not solved by routing:** an attacker controlling every source or the entire conditioner; direct extraction of private internal state; malicious firmware throughout the trusted computing base; a broken hash/KDF primitive; physical compromise beyond assumed protections; formal device-independent quantum randomness certification. No assertion of immunity to timing or power analysis is made.

**Failure policy:** health-test failure marks a source unavailable and triggers a visible audit event. Output must stop when fewer than the policy-mandated sources or a defensible entropy quota remain. The implementation must not silently replace faulty quantum measurements with deterministic bytes while maintaining a claim of QRNG-derived entropy.

## 6. Health monitoring and validation

NIST SP 800-90B describes entropy-source validation, including raw-noise-source health tests. Apply tests to the *individual unconditioned source streams* before combining them; evaluate start-up, continuous operation, restart behavior, and environmental range. In particular, assess the applicability and parameters of the Repetition Count Test and Adaptive Proportion Test to each noise-source model. Do not treat a generic post-conditioning battery, e.g., SP 800-22, as a substitute for source entropy estimation or health testing.

NIST SP 800-90C (final September 2025) covers RBG constructions and must be consulted separately before any claim that a system complies with the SP 800-90 family. Being *informed by* these standards is not equivalent to certification or validation.

### 6.1 Experimental comparison (preregister before testing)

| Design | Pooling | Read schedule | Matched source budget |
|---|---|---|---|
| B0 | One QRNG | Fixed | No; single-source reference |
| B1 | Multiple QRNGs | Fixed round-robin | Yes |
| B2 | Multiple QRNGs | Pseudorandom schedule seeded independently | Yes |
| UAQDR | Multiple QRNGs | Feedback-derived schedule from prior QRNG inputs | Yes |

Control for aggregate entropy input, byte count, source hardware, test environment, conditioner, output rate, and health-test thresholds. Compare throughput, median and tail latency, source-fault recovery, false alarm rate, sample loss, energy, and trace-based leakage. RQ3 needs explicit observer capabilities and an empirical attack/defense evaluation rather than visual claims about scrambled traces.

### 6.2 Necessary evidence before a stronger claim

- Measurement-device and QRNG physical entropy model, including classical noise contributions.
- Raw capture files with documented provenance and independent, reproducible evaluation.
- Per-source and conditional entropy estimates with declared assumptions and confidence intervals.
- Demonstrated source independence or conservatively revised bounds for correlated outputs.
- Adaptive path-bias and source-starvation tests, including injected source failures.
- Independent review of routing implementation and conditioning/KDF choices.
- Side-channel comparisons on matched hardware; report cases where dynamic routing hurts.
- Exact versioned configuration, source code, and negative or inconclusive results.

## 7. Engineering considerations

**Latency:** sequential hardware reads can increase worst-case delay; a bounded epoch and parallel sampling where physically possible are preferable. **Reproducibility:** secret scheduling seeds complicate trace diagnosis; preserve metadata without logging raw entropy, derived keys, or other sensitive internal state. **Hardware independence:** manufacturers, clock trees, supply rails, drivers, and sensors can introduce common-mode failures. **Security boundary:** the QRNG controller, integrity checks, memory scrubbing, boot process, and host API all warrant scrutiny. **Energy/cost:** adding generators is not free; the correct comparison is against a validated, simpler RBG with equivalent entropy assurance.

## 8. Scope of possible applications

Research prototypes, secure provisioning workflows, and systems where dedicated physical entropy collection and auditability justify extra complexity may be suitable *hypothetical* applications. The architecture is **not** a standalone encryption scheme, a replacement for accepted DRBG mechanisms, or proven better than existing certified hardware security modules. Do not deploy for real key material until independent validation and threat-model-specific security review are complete.

## 9. Conclusion and falsifiable hypothesis

UAQDR proposes an observable design difference—QRNG-derived feedback that governs future multi-source acquisition. The testable hypothesis is that, under specific device-fault or external-observer models and equal entropy budgets, it yields improved resilience or reduced information leakage without unacceptable latency. The null hypothesis is that matched static or independently randomized routing achieves equal or better results. Both outcomes are publishable research findings. This v0.1 draft provides a foundation for experiments, **not** proof of novelty or cryptographic effectiveness.

## References

1. NIST, *Recommendation for the Entropy Sources Used for Random Bit Generation*, SP 800-90B (2018). https://doi.org/10.6028/NIST.SP.800-90B
2. NIST, *Recommendation for Random Bit Generator (RBG) Constructions*, SP 800-90C (2025). https://doi.org/10.6028/NIST.SP.800-90C
3. NIST, *Recommendation for Random Number Generation Using Deterministic Random Bit Generators*, SP 800-90A Rev. 1 (2015). https://doi.org/10.6028/NIST.SP.800-90Ar1
4. N. Ferguson, B. Schneier, T. Kohno, *Cryptography Engineering*, chapter on Fortuna. Reference excerpt: https://www.schneier.com/wp-content/uploads/2015/12/fortuna.pdf
5. X. Ma et al., “Postprocessing for quantum random-number generators: Entropy evaluation and randomness extraction,” *Physical Review A* 87, 062327 (2013). https://doi.org/10.1103/PhysRevA.87.062327
6. S. Coretti et al., “On Seedless PRNGs and Premature Next,” *ITC 2022*. https://doi.org/10.4230/LIPIcs.ITC.2022.9
7. Related hardware random-routing study, “Multi-bit true random number generator design using dual-mode entropy sources and torus network architecture” (2025/2026). https://www.sciencedirect.com/science/article/pii/S1879239125004096 — **verify full bibliographic details before final publication**.

---

### Pre-publication checklist for author

- [ ] Verify each bibliographic entry and complete related-work search.
- [ ] Review the proposed formalism with an experienced cryptography / QRNG researcher.
- [ ] Distinguish unimplemented design decisions from experimentally supported results.
- [ ] Decide authorship, licensing, reproducibility/data-sharing, and any conflicts of interest.
- [ ] Convert to a searchable, properly typeset PDF with title, author, abstract, references.
- [ ] Publish a versioned preprint; add the real DOI only after reservation/publication.

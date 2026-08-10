# Quantum_algos

**A hands-on lab notebook exploring the transition from classical to post-quantum cryptography** — implemented from first principles, benchmarked against a real dataset, and grounded in a working demonstration of *why* the transition is necessary.

This repository is a set of self-contained Jupyter notebooks that (1) simulate the quantum threat to today's public-key cryptography, (2) baseline classical cryptographic primitives against a real-world tabular dataset, and (3) build up NIST-selected and NIST-adjacent post-quantum constructions — lattice-based encryption, hash-based signatures, and a compact lattice signature scheme — largely from the underlying math rather than by calling a black-box library.

It is not a production cryptography toolkit, and it isn't meant to be one. It's a record of working through the mechanics of PQC and QKD closely enough to reason about trade-offs, not just cite them.

---

## Why this exists

Shor's algorithm, run on a sufficiently large fault-tolerant quantum computer, breaks the integer-factorization and discrete-log problems that RSA, Diffie-Hellman, and ECC rely on. NIST responded with a multi-year process to standardize replacements, culminating in **FIPS 203 (ML-KEM / Kyber)**, **FIPS 204 (ML-DSA / Dilithium)**, and **FIPS 205 (SLH-DSA / SPHINCS+)**, with **FALCON** following as a compact-signature standard. Separately, **Quantum Key Distribution (QKD)** offers a physics-based (rather than computationally-hard-problem-based) approach to key agreement.

Rather than reading about this, this repo works through it directly:

- **Break RSA with Shor's algorithm** against a real (small) RSA keypair, to make the threat concrete instead of theoretical.
- **Simulate BB84 QKD** to see how eavesdropping is physically detectable, not just computationally hard to pull off.
- **Implement a lattice-based encryption scheme (LWE)** from raw linear algebra to understand *why* lattice problems resist quantum attack.
- **Build on the reference implementation of FALCON** (a NIST-selected lattice signature scheme) and wrap it around a practical use case (signing/encrypting tabular data).
- **Reconstruct core building blocks of SPHINCS+** (its Haraka-based hashing) to understand hash-based signatures, the most conservative security category in the NIST portfolio.
- **Baseline it all against classical crypto** (RSA-OAEP, AES-256-CBC, SHA-256) applied to an actual fraud-detection dataset, with timing and unit tests, so the PQC work has something concrete to be compared against.

---

## Repository contents

| File | Category | What it does |
|---|---|---|
| `RSA_PQC.ipynb` + `RSA_module.py` | **Quantum threat demo** | Generates a textbook RSA keypair, encrypts a message, then recovers the plaintext *without* the private key using a Qiskit implementation of Shor's period-finding routine. |
| `QKD_msg_UI.ipynb` | **QKD** | BB84 protocol simulation (Qiskit) for exchanging a text message: qubit prep, basis reconciliation, key sifting, and one-time-pad-style encryption of the message with the resulting key. Interactive `ipywidgets` UI. |
| `QKD_coord_UI.ipynb` | **QKD** | Same BB84 mechanics, applied to encrypting numeric coordinate data instead of text — demonstrates the protocol generalizes to arbitrary payloads once you have a shared bitstring. |
| `Lattice_cryptography(learning with errors).ipynb` | **Post-quantum (lattice)** | A from-scratch NumPy implementation of single-bit LWE (Learning-With-Errors) encryption — the hard problem underlying ML-KEM/Kyber — including key generation, encryption, and decryption. |
| `Falcon.ipynb` | **Post-quantum (lattice signatures)** | Builds on the official Python reference implementation of **FALCON** (NTRU-lattice, hash-and-sign, Fast-Fourier sampling) and wraps it in a `FalconDataFrameHandler` class for signing/encrypting pandas DataFrames. |
| `Sphincs_pl_Haraka.ipynb` | **Post-quantum (hash-based signatures)** | Reconstructs the **Haraka** short-input hash function used inside SPHINCS+, and applies it to hash a tabular dataset cell-by-cell for integrity verification. |
| `Isogeny_cryptography(RSA-OAEP+AES-GCM).ipynb` | **Hybrid encryption + benchmarking** | The most complete notebook: RSA-OAEP + AES-GCM hybrid encryption of a dataset, a full `unittest` suite (including negative tests — tampered ciphertext, wrong key, invalid input), and empirical runtime-vs-dataset-size plots (linear and log-log). See note below on the naming. |
| `Asymmetric_key_crypto(RSA-OAEP).ipynb` | **Classical baseline** | RSA-OAEP + AES hybrid encryption applied column-wise to a fraud-detection dataset, timed with `%%time`. |
| `Symmetric_key_cryptography(AES-256-CBC with PKCS7 padding).ipynb` | **Classical baseline** | AES-256-CBC with PKCS7 padding, same dataset and methodology, for symmetric-key comparison. |
| `Hash_cryptography(SHA-256).ipynb` | **Classical baseline** | Cell-level SHA-256 hashing of the dataset plus a verification routine, as an integrity-check baseline to compare against Haraka. |

---

## The quantum-threat demo, in a bit more detail

<details>
<summary><strong>How <code>RSA_PQC.ipynb</code> breaks RSA (click to expand)</strong></summary>

<br>

1. `RSA_module.py` generates a small textbook RSA keypair — deliberately unoptimized (naive `gcd`, brute-force `mod_inverse`, trial-division primality) so the modulus stays small enough for a quantum *simulator* to attack in reasonable time.
2. A message is encrypted with the public key.
3. `shors_breaker()` uses Qiskit to run a period-finding subroutine (the quantum part of Shor's algorithm — finding the order `r` of `a mod N`), then uses that period classically via `gcd(a^(r/2) ± 1, N)` to recover the prime factors `p, q`.
4. From `p` and `q`, Euler's totient and the private exponent `d` are reconstructed, and the ciphertext is decrypted — with no access to the original private key.

This is a **simulated, small-N demonstration** (the real algorithm needs a fault-tolerant quantum computer with far more logical qubits than exist today to threaten real 2048-bit RSA). Its value here isn't scale — it's making the *mechanism* of the threat tangible: factoring falls out of period-finding, and period-finding is the part a quantum computer can do exponentially faster than a classical one.

</details>

---

## A note on honesty about scope

A few things worth being upfront about, since they matter for how this code should be read:

- **`Isogeny_cryptography...ipynb` doesn't implement isogeny-based cryptography.** Despite the filename, it's a classical RSA-OAEP + AES-GCM hybrid. (SIDH/SIKE, the primary isogeny-based NIST candidate, was cryptanalytically broken via a classical attack in 2022 and withdrawn from the process — so there wasn't a maintained, safe isogeny library to build on. The notebook is best read as a hybrid-encryption benchmarking exercise, not an isogeny implementation.)
- **`Sphincs_pl_Haraka.ipynb` uses randomly generated round constants**, not the official Haraka constants from the SPHINCS+ specification. It reproduces the *structure* of Haraka (AES-round-based permutation, sponge-style absorption) for learning purposes — it is not spec-compliant and shouldn't be treated as an interoperable implementation.
- **`Falcon.ipynb` depends on the official FALCON Python reference implementation** (`common`, `fft`, `ntt`, `ffsampling`, `ntrugen`, `encoding`, `rng` modules from the [falcon-sign.info](https://falcon-sign.info/) reference code) — these files are not vendored into this repo and need to be placed alongside the notebook to run it.
- **The classical-baseline notebooks expect Kaggle-style fraud datasets** (`fraudTrain.csv`, `creditcard.csv`) that aren't included here for size/licensing reasons — see Setup below.
- **None of this is audited or intended for production use.** Several files (e.g. `RSA_module.py`'s primality check, or plaintext-adjacent handling in early cells) are intentionally simple for pedagogical clarity, not hardened for real deployment.

I'd rather the README say this plainly than have someone assume otherwise from the filenames.

---

## Tech stack

| Layer | Tools |
|---|---|
| Quantum simulation | [Qiskit](https://www.ibm.com/quantum/qiskit) + Qiskit Aer (`QasmSimulator`) |
| Classical crypto primitives | [`cryptography`](https://cryptography.io/) (`hazmat` primitives: RSA-OAEP, AES-CBC/PKCS7, SHA-256), [PyCryptodome](https://www.pycryptodome.org/) (`Crypto.PublicKey`, `Crypto.Cipher`) |
| Post-quantum reference code | Official FALCON Python reference implementation (NTRU lattices, FFT sampling) |
| Numerical / data | NumPy, pandas, scikit-learn (`train_test_split`, `StandardScaler`, `metrics`) |
| Visualization | Matplotlib |
| Interactive UI | `ipywidgets` (for the QKD notebooks) |
| Testing | Python `unittest` (positive + negative/adversarial cases in the hybrid-encryption notebook) |

---

## Setup

```bash
# Python 3.9–3.10 recommended (Qiskit's older Aer API used here doesn't play well with 3.11+)
python -m venv venv && source venv/bin/activate

pip install qiskit "qiskit-aer<0.13" ipywidgets \
            cryptography pycryptodome \
            numpy pandas scikit-learn matplotlib jupyter
```

**Notes:**
- The QKD notebooks use `qiskit.providers.aer` and the `execute()` function, which were removed in Qiskit 1.0+. Pin to a **Qiskit 0.x** environment (`qiskit<1.0`) to run them as written, or port `execute(...)` calls to `Aer.get_backend(...).run(...)` for a modern install.
- `Falcon.ipynb` needs the FALCON reference implementation's supporting modules (`common.py`, `fft.py`, `ntt.py`, `ffsampling.py`, `ntrugen.py`, `encoding.py`, `rng.py`) in the same directory — available from the [official FALCON site](https://falcon-sign.info/).
- The classical-baseline notebooks (`Asymmetric_key_crypto`, `Symmetric_key_cryptography`, `Hash_cryptography`, `Isogeny_cryptography`) expect a `fraudTrain.csv` / `creditcard.csv` in the working directory — any Kaggle credit-card-fraud dataset with a `Class` column will work as a drop-in.

---

## What this demonstrates

- Comfort moving between **the theory NIST standardizes** (lattice problems, hash-based signatures, KEMs) and **the code that implements it**, rather than only knowing algorithm names.
- An understanding that PQC migration isn't a single algorithm swap — it spans **encryption (KEMs), signatures, and hashing**, each with different security assumptions and NIST-selected candidates (ML-KEM, ML-DSA, SLH-DSA, FALCON).
- Practical exposure to **QKD as an alternative key-agreement mechanism**, distinct from PQC's "use a different hard problem" approach.
- A working, if simplified, mental model of **why** Shor's algorithm forces this migration — through an actual break, not just the claim.
- Willingness to **benchmark and test** (the hybrid-encryption notebook's `unittest` suite and runtime-scaling plots), and to be transparent about where an implementation is a simplification, a proxy, or incomplete — which matters more in cryptography than in most other software.

---

## References

- NIST Post-Quantum Cryptography Project — <https://csrc.nist.gov/projects/post-quantum-cryptography>
- FIPS 203 (ML-KEM), FIPS 204 (ML-DSA), FIPS 205 (SLH-DSA) — NIST, 2024
- FALCON specification — <https://falcon-sign.info/>
- Bennett & Brassard, *Quantum Cryptography: Public Key Distribution and Coin Tossing* (BB84), 1984
- Castryck & Decru, *An Efficient Key Recovery Attack on SIDH*, 2022 (the SIKE break referenced above)
- Shor, P., *Algorithms for Quantum Computation: Discrete Logarithms and Factoring*, 1994

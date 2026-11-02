# Hack the Chip 2026 — Autonomous NKI Kernel Agent

**NYU × Amazon Annapurna Labs | Team project | Qwen3-8B, AWS Trainium2, Python, Neuron Kernel Interface (NKI)**

This is a short engineering case study for readers evaluating the [actual Project 2 implementation](./README.md). The [repository's default branch](../../README.md) contains the workshop scaffold; the team's work is on the **feat/nki-diagnostic-selection** development line. The original workshop materials preceded this team implementation.

> **What it is:** An agent that iteratively proposes NKI kernels, repairs a bounded set of hardware-API mistakes, and tests generated code against an independent simulator-based correctness checker. Its goal is correct kernel generation and failure diagnosis, not a claimed hardware speedup.

## The problem

General-purpose language models can produce plausible-looking code for specialized accelerator APIs that does not compile, exceeds tensor tiling limits, uses invalid memory buffers, or gives incorrect numerical results. Repeated free-form prompting is unreliable unless failures are classified and translated into actionable feedback.

The project explores a controlled **generate → check → diagnose → repair** loop using Qwen3-8B served on Trainium2.

## System design

~~~mermaid
flowchart LR
  T[Task + official shapes] --> P[Hardware-aware planner]
  SDK[Installed NKI SDK] --> P
  P --> M[Qwen3-8B candidates]
  M --> L[Conservative AST legalizer]
  L --> C[Independent NKI checker]
  C -->|verified| V[Save locked candidate + hash]
  C -->|not verified| D[Classify failure + select candidate]
  D -->|targeted feedback| M
~~~

| Engineering component | Why it matters | Source |
| --- | --- | --- |
| Hardware-aware task planner | Builds compact hints using task shapes, output contracts, memory limits and installed API signatures | [kernel_planner.py](./kernel_planner.py) |
| Kernel generation loop | Proposes candidates, tracks attempts, and terminates only on checker acceptance | [agent.py](./agent.py) |
| AST-based primitive legalizer | Repairs narrow, known NKI call/namespace/buffer mistakes; every change still requires verification | [primitive_legalizer.py](./primitive_legalizer.py) |
| Independent checker | Checks static rules, simulation output, input integrity, HBM traffic and reported hardware hazards | [nkibench.py](./nkibench.py) |
| Candidate/diagnosis selection | Preserves the highest reward; uses failure evidence as a tie-break or repair cue | [failure_selection.py](./failure_selection.py) |
| Controlled experiments | Isolates experiment runs, preserves manifests and records candidate outcomes | [run_controlled.py](./run_controlled.py) |
| Offline evaluation | Includes deterministic tests and synthetic error/repair examples | [tests](./tests), [synthetic_nki](./synthetic_nki) |

A standard budget in the team's README is eight rounds with four candidates per round. The agent records attempts and the reasons for accepting or rejecting kernels.

## Reported results — with necessary context

The [team's project README](./README.md#results) reports the following **simulator** outcomes:

| Level | NKI task | Reported reward | Official shapes |
| --- | --- | ---: | ---: |
| 1 | Average pooling (warm start) | 1.00 | 4/4 |
| 2 | Transpose | 1.00 | 4/4 |
| 3 | Single-tile matrix multiplication | 1.00 | 1/1 |
| 4 | Tiled matrix multiplication | 1.00 | 4/4 |
| 5 | Matrix multiplication with hoisted loads | 1.00 | 4/4 |
| 6 | M/N blocked matrix multiplication | 1.00 | 4/4 |
| 7 | M/N/K blocked matrix multiplication | 1.00 | 4/4 |
| 8 | Single-head attention | Not completed in the reported summary | — |

**Interpretation:** In the team's reported runs, seven of eight levels reached the checker acceptance reward, with a **warm-start** qualification for Level 1. These individual successes must **not** be described as a 7/8 cold-start solve rate across independent runs. Earlier ablations had lower results, and the [submission notes](../../HACKATHON_SUBMISSION.md) document failures and the limits of the evidence.

The submission notes also report:
- A **small, skewed synthetic** failure/repair corpus with bug-specific checks of **21/38 (legacy)** versus **30/38 (targeted)**; these are not general model-accuracy claims.
- Qwen3-8B LoRA training for 82 steps on 41 examples, with held-out loss falling from 0.9951 to 0.1275. **No demonstrated improvement on the original kernel benchmark is established by that loss change.**

These figures reflect historical team documentation. This case study **does not claim an independent rerun**, production deployment, hardware kernel execution, or device-level latency/speedup measurements.

## How to inspect or reproduce

Start with the [detailed implementation README](./README.md). The original experiment requires the NKI SDK and the relevant Trainium/Neuron setup and language-model endpoint. A representative check from the project directory is:

~~~bash
python -B nkibench.py --selftest
python -B -m pytest tests -q
~~~

The full agent and controlled experiment commands, including source isolation, grading directories and logging, are in the [project README](./README.md#quick-start-on-your-trainium-seat). They are not expected to run on an ordinary laptop without the required NKI environment.

## Engineering lessons

1. **Correctness needs a checker.** A model's confident code is not evidence that an NKI kernel is valid.
2. **Feedback must be actionable.** Reporting unsupported instructions or memory placement failures is useful only when the next attempt can address them.
3. **Diagnostics and legalization are bounded.** A generic AST fix is not a proof of algorithmic correctness; independent numerical checks remain authoritative.
4. **Experimental controls matter.** Prompt settings, sampling policies, warm starts and repeated runs can change outcomes. Separate a verified best candidate from repeatable cold-start success.

## Scope, credit and portfolio usage

This is **collaborative hackathon work** in a teammate-maintained repository, built upon public workshop materials. The feature branch's commits and the original workshop history document provenance; this case study does not attempt to assign individual authorship of all modules.

A defensible résumé description is:

> Contributed to a team-built autonomous NKI kernel-generation and repair agent for AWS Trainium2 using Qwen3-8B, hardware-aware prompting, AST-based API legalization and simulator-backed verification; documented checker-accepted solutions for seven challenge levels under reported configurations.

That wording reflects the reported outcomes **without** implying sole authorship or measuring hardware acceleration. Anyone using this case study on a portfolio should link directly to the project's development branch and clarify their own contributions during interviews.

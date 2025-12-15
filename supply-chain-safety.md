# ArcSight – Supply Chain & Execution Safety



This document describes what happens when ArcSight is executed.



## Execution Model



- ArcSight performs **read-only analysis**

- No files are modified

- No code is generated

- No network requests are made during analysis



## Network Behavior



- ArcSight does **not** make outbound network calls

- No telemetry is sent

- No data leaves the machine



## Installation Options



You do not need to use `npx`.



Alternatives:

- Clone the repository and build locally

- Inspect the source before execution

- Vendor the binary if preferred



## Determinism



- Output is deterministic and reproducible

- No timestamps, randomness, or environment-dependent behavior

- Same input produces the same output byte-for-byte



## Auditing



- Open source

- No obfuscation

- All execution paths are visible in code



## Failure Behavior



- If confidence is insufficient, ArcSight produces no output

- Silence is intentional and documented



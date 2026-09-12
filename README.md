# AI-Powered Intrusion Detection System (IoT / RPL Routing Attacks)

A machine learning project that classifies IoT network traffic as **Normal** or one of four
**RPL routing-protocol attacks** — Blackhole, Flooding, Rank, and Version — and, more
importantly, documents the real engineering process behind it: a data leakage bug that was
found and fixed, an honest comparison across a statistical baseline and two stronger models,
and a diagnosis of *why* accuracy plateaus where it does.

This README is written to walk through that whole process, not just report a final number.

---

## 1. The Problem

**RPL** (Routing Protocol for Low-power and Lossy Networks) is how IoT devices build their
routing tree. Several well-known attacks target it:

| Attack | What it does |
|---|---|
| **Blackhole** | A malicious node silently drops all traffic routed through it |
| **Flooding** | Overwhelms the network with excessive control messages |
| **Rank** | A node lies about its position in the routing tree to attract traffic |
| **Version** | Forces unnecessary, costly network-wide topology rebuilds |

**Goal:** given a single simulated RPL control message, predict whether it's Normal traffic
or one of these four attacks — a **5-class supervised classification** problem.

---

## 2. The Dataset

**[IoT-RPL 2021: Cyber Attack Dataset Based on RPL Routing for IoT](https://data.mendeley.com/datasets/4rcbbry2sc/1)**
— Walid Dhifallah, Mounira Tarhouni, Tarek Moulahi, Salah Zidi. Mendeley Data, DOI:
[10.17632/4rcbbry2sc.1](https://doi.org/10.17632/4rcbbry2sc.1), licensed CC BY 4.0.

Published specifically to support IDS research on RPL-based IoT/6LoWPAN networks so
researchers don't have to simulate the attacks themselves. Stored locally as
`Dataset/RPL_Routing_Attacks.csv` — one row per RPL control message (DIS / DIO / DAO).

> **Scope note:** the full published dataset is split across 10 files (`0.csv` through
> `9.csv`, **10,242,176 rows total**). For this project, only **one file (`0.csv`,
> 1,048,575 rows — ~10% of the full dataset)** was used — a deliberate scope decision to
> keep training times and iteration speed reasonable for a first version, not a limitation
> of the approach itself. The full 10-file set would be the natural next step to test
> whether findings (e.g. the Blackhole/Version overlap in Section 6) hold at larger scale.

- **1,048,575 rows** (from the one file used), 23 columns (22 features + `label`)
- Class distribution (imbalanced — see below):

| Label | Count |
|---|---|
| Flooding | 365,603 |
| Normal | 219,470 |
| Blackhole | 185,749 |
| Version | 147,224 |
| Rank | 130,529 |

Columns include protocol layers (`frame_proto`, `protocol`, `control_type`), the RPL message
type (`type_cont_messg`: DIS/DIO/DAO), and protocol-specific fields (`DOAGID`, `DIO_info`,
`object_cont_pt`, etc.).

---

## 3. Preprocessing Pipeline

Real-world data isn't clean — several columns mix numbers, hex codes, and comma-separated
lists depending on the RPL message type. For every feature column, the pipeline:

1. **Detects list-type columns** (e.g. `to`, a comma-separated neighbor list like `"4,10,11"`)
   → converts to a **neighbor count** instead.
2. **Tries numeric conversion** (`pd.to_numeric`) — if 90%+ of a column converts cleanly, it's
   treated as a real number (gaps filled with 0).
3. **Otherwise, treats it as categorical** and applies **Label Encoding** (text → integer ID).
4. The target label (`Normal`/`Blackhole`/.../`Version`) is also **Label Encoded** into a plain
   integer — no further transformation needed, since every model used here (Logistic
   Regression, Random Forest, Gradient Boosting) works directly with integer labels.
5. **Scaling**: `StandardScaler` normalizes every feature to a comparable range (protocol
   fields like `DOAG_info` reach into the billions; others are 0/1 flags).
6. **Train/test split**: 80/20, **stratified** to preserve class proportions given the imbalance.

Full step-by-step version, with explanations: [`model.ipynb`](model.ipynb).

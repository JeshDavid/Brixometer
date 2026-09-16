<div align="center">

# 🌾 Brixometer

**A CryoCivic product.** Lab-grade sugarcane intelligence in your hand — and fairness at the mill gate.

Patent-pending handheld AIoT instrument (Indian Patents Act, 1970 — Form 2
Complete Specification filed) that measures sugarcane sucrose non-destructively
in the field, and reconstructs *true harvest Brix* at the mill gate after
transit degradation, with a hard integrity gate on every mill-side correction.

[![CI](https://github.com/JeshwinDavid/brixometer/actions/workflows/ci.yml/badge.svg)](https://github.com/JeshwinDavid/brixometer/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Python 3.11+](https://img.shields.io/badge/python-3.11%2B-blue.svg)](https://www.python.org/downloads/)
[![Platform: ESP32-S3](https://img.shields.io/badge/platform-ESP32--S3-e7352c.svg)](https://www.espressif.com/en/products/socs/esp32-s3)
[![TRL 4](https://img.shields.io/badge/TRL-4%20%7C%20validated%20in%20lab-orange.svg)](docs/ROADMAP.md)
[![Patent pending](https://img.shields.io/badge/status-patent%20pending-8a2be2.svg)](docs/PATENT.md)

[My role](#-my-role-on-this-project) ·
[Overview](#-overview) ·
[How it works](#-how-it-works) ·
[Quick start](#-quick-start) ·
[Patent mapping](docs/PATENT.md) ·
[Science](docs/SCIENCE.md)

<img src="docs/assets/device_render.jpg" alt="CryoCivic Brixometer device render" width="440">

</div>

---

> [!IMPORTANT]
> **Brixometer is at TRL 4 — validated in laboratory, not in the field.**
> Every performance figure here is a *target* or a lab result on a controlled
> 200-stem, 4-variety dataset. See [`docs/ROADMAP.md`](docs/ROADMAP.md) for
> exactly what still needs field evidence.

---

## 👤 My role on this project

Brixometer is a multi-inventor, patent-pending hardware+software project
(full applicant list and claim mapping in [`docs/PATENT.md`](docs/PATENT.md)).
**This repository is the software half, and it's mine end to end:**

- **Edge ML** — a Conformal Mixture-of-Experts regressor (`ml/cmoe_model.py`)
  with split-conformal calibration (`ml/conformal.py`) that gives every
  prediction a distribution-free uncertainty interval, quantised to int8 and
  exported for on-device inference.
- **Embedded firmware (C++/ESP32-S3)** — sensor drivers with dark/white
  referencing, a contact-force + orientation gate that refuses a capture
  before it's taken (`firmware/src/contact_gate.cpp`), and an on-device
  int8 inference runtime with byte-identical preprocessing to the training
  pipeline — checked by dedicated host/device parity tests, not assumed.
- **Applied domain science** — Arrhenius inversion kinetics for post-harvest
  sucrose loss, with a QR/TTI cross-validation gate that will *refuse* to
  correct a payment-affecting reading rather than guess when the two signals
  disagree (`ml/arrhenius_tti.py`, mirrored in `firmware/src/tti_decode.cpp`).
- **Backend/API** — a FastAPI lot registry (`backend/app.py`) that issues QR
  codes, stores signed grades *and signed refusals*, and exposes `/verify` so
  a disputed grade — or a disputed refusal — can be independently recomputed.
- **Test discipline** — 46 unit tests covering kinetics, conformal coverage,
  the integrity gate's refusal paths, and host/device numerical parity,
  running in CI on every push.

**Why this is the project I'd point to for a software engineering internship
at an ER&D company like Tata Elxsi:** it isn't a toy. It's a real, filed
system with hard real-time constraints (sub-3-second field decisions), a
payment-affecting correctness requirement (the mill-side gate), a
resource-constrained target (int8 inference on a microcontroller), and a
documented boundary between what's proven and what isn't. That combination —
embedded systems + edge AI + systems-level correctness thinking — is close to
the actual shape of ER&D work.

## 📋 Overview

| | |
|---|---|
| **Product** | CryoCivic / Brixometer |
| **Domain** | Precision Agriculture (AgriTech) |
| **Sub-domains** | Edge AI/ML · NIR Spectroscopy & Sensor Fusion · IoT & Embedded · Post-Harvest Food Science · Supply Chain Integrity |
| **Maturity** | TRL 4 — validated in laboratory |
| **IP status** | Patent application filed (Form 2, Indian Patents Act 1970) |
| **Target price** | ₹22,000 (≈2% of a benchtop lab NIR) |
| **Core stack** | ESP32-S3 · TFLite Micro · PyTorch · FastAPI |

## 🔴 The problem

India's sugarcane sector supports **50M+ farmers** and still runs on guesswork.

**In the field.** Farmers decide when to cut using leaf colour, the season
calendar, and a neighbour's opinion. Cut early and sucrose hasn't peaked; cut
late and the crop inverts and attracts pests — costing **₹5,000–18,000 per
season** per smallholder.

**At the mill gate.** Cane loses sucrose in transit through heat-driven
inversion — irreversible, and outside the farmer's control. Mills grade and
pay on *arrival* quality, not *harvest* quality, with no traceability and no
correction mechanism.

## 💡 How it works

One measurement head, one bayonet-mounted adapter the user swaps by hand:
a **conical nose** for a standing stem in the field, a **flat-faced tip** for
a cut billet at the mill. A mode button tells the firmware which pipeline to
run on the same sensor data.

### 🟢 Field Mode — *"should I cut today?"*

```
Contact-force + orientation gate (must pass before capture is even taken)
    → 18-ch NIR reflectance + cross-polar RGB
    → Conformal Mixture-of-Experts (int8, on-device)
    → Brix estimate + calibrated uncertainty interval
    → GREEN / YELLOW / RED badge
```

| Badge | Meaning |
|:--:|---|
| 🟢 **GREEN** | Whole interval in the harvest window — cut now |
| 🟡 **YELLOW** | Interval straddles a threshold — device declines to guess. Rescan |
| 🔴 **RED** | Confidently outside the window — still maturing, or over-ripe |

> [!NOTE]
> A false GREEN costs a farmer a season of sucrose; a false YELLOW costs them
> thirty seconds. The decision rule is deliberately biased toward abstention,
> and `evaluate.py` reports the false-GREEN rate as a separate, blocking metric.

### 🏭 Mill Mode — *"what was this lot worth at harvest?"*

```
Scan lot QR (declared harvest timestamp) + TTI thermal strip (measured exposure)
    → do the two independently agree, within tolerance and within a trusted
      exposure bound?
         NO  → correction disabled. Signed record flags the lot for manual QA.
         YES → Arrhenius backcast: S_harvest = S_arrival · exp(D)
             → Ed25519-signed, auditable grade
```

This is the part of the system I'd most want a reviewer to look at closely:
**the correction is a privilege the reading has to earn, not a default.**
See [`ml/arrhenius_tti.py::gated_backcast_from_tti()`](ml/arrhenius_tti.py) and
its firmware mirror in [`firmware/src/tti_decode.cpp`](firmware/src/tti_decode.cpp) —
both return a signed *refusal*, never a guessed number, when the QR-declared
transit time and the TTI-implied transit time disagree beyond tolerance, or
when declared exposure exceeds the bound the model is trusted for.

<div align="center">
<img src="docs/assets/patent_fig1_isometric.jpg" alt="Patent Fig. 1 - isometric view" width="360">
<img src="docs/assets/patent_fig2_exploded.jpg" alt="Patent Fig. 2 - exploded view" width="360">
<br><sub>Figures from the filed Form 2 Complete Specification — full reference numeral list in <a href="docs/PATENT.md">docs/PATENT.md</a></sub>
</div>

## 🗂 Repository layout

```
brixometer/
├── 📁 docs/              Science, architecture, patent mapping, roadmap, pitch copy
├── 📁 firmware/          ESP32-S3 (PlatformIO)
│   ├── include/          Config, pin map, generated model header
│   └── src/              Sensor drivers, contact gate, CMoE runtime,
│                         TTI integrity gate, signing, UI
├── 📁 ml/                Training pipeline (PyTorch) → TFLite Micro export
│   ├── preprocessing.py  SNV · Savitzky-Golay · referencing
│   ├── cmoe_model.py     Gated Mixture-of-Experts regressor
│   ├── conformal.py      Split-conformal calibration → prediction intervals
│   ├── arrhenius_tti.py  Inversion kinetics + gated TTI backcasting
│   ├── train.py          Leave-one-variety-out CV + calibration split
│   ├── evaluate.py       RMSE · R² · SEP · RPD · conditional coverage
│   └── export_tflite.py  int8 export → C array for firmware
├── 📁 backend/           FastAPI lot registry — QR issue, custody, signed grades and refusals
├── 📁 hardware/          BOM, optical path, enclosure notes
├── 📁 scripts/           Synthetic dataset generator
└── 📁 tests/             46 tests — kinetics, coverage, integrity gate, host/device parity
```

## 🚀 Quick start

### ML pipeline

```bash
git clone https://github.com/<your-username>/brixometer.git
cd brixometer
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

python scripts/make_synthetic_dataset.py --n 600 --out ml/data/raw/synthetic.csv
python ml/train.py --data ml/data/raw/synthetic.csv --out ml/artifacts
python ml/evaluate.py --artifacts ml/artifacts --data ml/data/raw/synthetic.csv
python ml/export_tflite.py --artifacts ml/artifacts --header firmware/include/cmoe_model.h
```

> [!WARNING]
> Synthetic data is scaffolding for smoke-testing the pipeline. No result
> computed from it is a validation result. Schema for real data:
> [`docs/DATA_SPEC.md`](docs/DATA_SPEC.md).

### Backend

```bash
uvicorn backend.app:app --reload     # → http://127.0.0.1:8000/docs
```

Try the integrity gate directly:

```bash
curl -X POST localhost:8000/verify -H 'Content-Type: application/json' -d '{
  "arrival_brix": 16.5, "tti_response": 0.9, "declared_transit_hours": 1.0
}'
# -> {"status": "integrity_mismatch", "harvest_brix": null, ...}
```

### Firmware

```bash
cd firmware && pio run && pio run -t upload && pio device monitor
```

### Tests

```bash
pytest -q        # 46 passing
ruff check .
```

## ⚠️ Known limitations

Detail in [`docs/SCIENCE.md`](docs/SCIENCE.md). Stated plainly because a
project that names its own limits is worth more than one that overclaims.

- **This predicts a refractometer reading; it does not measure Brix directly.**
  Valid only inside the calibration set's range of varieties and conditions.
- **18 sparse channels is not a spectrum.** Expect lower accuracy than
  published benchtop cane-NIR results.
- **Conformal coverage is marginal, not conditional.** Per-variety
  undercoverage is a blocking issue and is reported separately.
- **First-order kinetics degrade past ~48 h transit.**
- **The Ed25519 signing routine is currently a stub** and must be replaced
  with hardware-backed key storage before any field unit ships.

## 🛣 Roadmap

TRL 4 (current, lab-validated) → Phase 1 field pilot (TRL 5) → Phase 2 mill
integration (TRL 6) → Phase 3 pre-production (TRL 7). Full exit criteria and
risk register: [`docs/ROADMAP.md`](docs/ROADMAP.md).

## 🤝 Contributing

`ruff check .` and `pytest` before a PR. Project-specific rules in
[`docs/CONTRIBUTING.md`](docs/CONTRIBUTING.md) — the short version: never
widen a claim beyond its evidence, host/device parity is a correctness
property, the device must be allowed to refuse, and the server never computes
a grade, only stores and verifies one.

## 📄 Licence

MIT — see [`LICENSE`](LICENSE). Patent application filed separately; see
[`docs/PATENT.md`](docs/PATENT.md).

---

<div align="center">

Built by **Jeshwin David C** · B.Tech AI & Data Science, Mar Ephraem College of
Engineering and Technology · Data Analyst Intern, Uproot Innovation

*Brixometer follows SSMA, and carries the same lesson forward: real problems
demand real domain knowledge, hardware and software must evolve together, and
honest teamwork always produces more reliable results than individual
brilliance.*

</div>

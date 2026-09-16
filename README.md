<div align="center">

# 🌾 Brixometer

**Lab-grade sugarcane intelligence in your hand — and fairness at the mill gate.**

A handheld AIoT instrument that measures sugarcane sucrose non-destructively in the field,
and reconstructs *true harvest Brix* at the mill gate after transit degradation.
One device. Two modes. One value chain.

[![CI](https://github.com/JeshwinDavid/brixometer/actions/workflows/ci.yml/badge.svg)](https://github.com/JeshwinDavid/brixometer/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Python 3.11+](https://img.shields.io/badge/python-3.11%2B-blue.svg)](https://www.python.org/downloads/)
[![Platform: ESP32-S3](https://img.shields.io/badge/platform-ESP32--S3-e7352c.svg)](https://www.espressif.com/en/products/socs/esp32-s3)
[![TRL 4](https://img.shields.io/badge/TRL-4%20%7C%20validated%20in%20lab-orange.svg)](docs/ROADMAP.md)
[![SDG 2 · 8 · 9](https://img.shields.io/badge/SDG-2%20%C2%B7%208%20%C2%B7%209-2ea44f.svg)](#-sustainable-development-impact)

[Overview](#-overview) ·
[How it works](#-how-it-works) ·
[Quick start](#-quick-start) ·
[Repository](#-repository-layout) ·
[Science](docs/SCIENCE.md) ·
[Roadmap](docs/ROADMAP.md)

</div>

---

> [!IMPORTANT]
> **Brixometer is at TRL 4 — validated in laboratory, not in the field.**
> Every performance figure in this repository is a *target* or a lab result on a
> controlled 200-stem, 4-variety dataset. Nothing here has been validated across
> seasons, operators or real mill logistics. See [`docs/ROADMAP.md`](docs/ROADMAP.md)
> for exactly what still needs evidence.

---

## 📋 Overview

| | |
|---|---|
| **Domain** | Precision Agriculture (AgriTech) |
| **Sub-domains** | Edge AI/ML · NIR Spectroscopy & Sensor Fusion · IoT & Embedded · Post-Harvest Food Science · Supply Chain Integrity |
| **Maturity** | TRL 4 — validated in laboratory |
| **Target price** | ₹22,000 (≈2% of a benchtop lab NIR) |
| **Core stack** | ESP32-S3 · TFLite Micro · PyTorch · FastAPI |
| **Licence** | MIT |

## 🔴 The problem

India's sugarcane sector supports **50M+ farmers** and still runs on guesswork.

**In the field.** Farmers decide when to cut using leaf colour, the season calendar,
and a neighbour's opinion. Cut early and sucrose hasn't peaked; cut late and the crop
inverts and attracts pests. Either way the loss lands on the farmer —
roughly **₹5,000–18,000 per season** per smallholder.

**At the mill gate.** Cane loses sucrose in transit through heat-driven inversion —
irreversible, and entirely outside the farmer's control. Mills grade and pay on
*arrival* quality, not *harvest* quality. The farmer is penalised for a truck queue
they did not create, with no traceability and no correction mechanism.

Precision today is a privilege of agribusinesses that can afford ₹10 Lakh+ laboratory
instruments. Brixometer closes that gap.

## 💡 How it works

### 🟢 Field Mode — *"should I cut today?"*

```
18-ch NIR reflectance + cross-polar RGB
    → Conformal Mixture-of-Experts (int8, on-device)
    → Brix estimate + calibrated uncertainty interval
    → GREEN / YELLOW / RED badge
```

No juice extraction. No refractometer. No lab. **~3 seconds per stem.**

| Badge | Meaning |
|:--:|---|
| 🟢 **GREEN** | Whole interval sits in the harvest window — cut now |
| 🟡 **YELLOW** | Interval straddles a threshold — the device declines to guess. Rescan |
| 🔴 **RED** | Confidently outside the window — still maturing, or over-ripe and inverting |

> [!NOTE]
> **The YELLOW badge is the most important feature in this project.** A false GREEN
> costs a farmer a season of sucrose; a false YELLOW costs them thirty seconds. The
> decision rule is deliberately biased toward abstention, and `evaluate.py` reports
> the false-GREEN rate as a separate, blocking metric.

### 🏭 Mill Mode — *"what was this lot worth at harvest?"*

```
Scan lot QR + TTI thermal strip
    → recover accumulated thermal dose  D = ∫ k(T(t)) dt
    → Arrhenius backcast:  S_harvest = S_arrival · exp(D)
    → Ed25519-signed, auditable grade record
```

The mill gets a defensible number. The farmer gets paid for what they actually grew.

## 🗂 Repository layout

```
brixometer/
├── 📁 docs/              Science, architecture, data spec, roadmap, pitch copy
├── 📁 firmware/          ESP32-S3 (PlatformIO)
│   ├── include/          Config, pin map, generated model header
│   └── src/              Sensor drivers, CMoE runtime, TTI decode, signing, UI
├── 📁 ml/                Training pipeline (PyTorch) → TFLite Micro export
│   ├── preprocessing.py  SNV · Savitzky-Golay · referencing
│   ├── cmoe_model.py     Gated Mixture-of-Experts regressor
│   ├── conformal.py      Split-conformal calibration → prediction intervals
│   ├── arrhenius_tti.py  Inversion kinetics + TTI backcasting
│   ├── train.py          Leave-one-variety-out CV + calibration split
│   ├── evaluate.py       RMSE · R² · SEP · RPD · conditional coverage
│   └── export_tflite.py  int8 export → C array for firmware
├── 📁 backend/           FastAPI lot registry — QR issue, custody, signed grades
├── 📁 hardware/          BOM, optical path, enclosure notes
├── 📁 scripts/           Synthetic dataset generator
└── 📁 tests/             41 tests — kinetics, coverage, preprocessing, host/device parity
```

## 🚀 Quick start

### ML pipeline

```bash
git clone https://github.com/<your-username>/brixometer.git
cd brixometer
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# Generate synthetic data so the pipeline runs out of the box
python scripts/make_synthetic_dataset.py --n 600 --out ml/data/raw/synthetic.csv

# Train → calibrate → evaluate
python ml/train.py --data ml/data/raw/synthetic.csv --out ml/artifacts
python ml/evaluate.py --artifacts ml/artifacts --data ml/data/raw/synthetic.csv

# Export int8 model + C header for the ESP32-S3
python ml/export_tflite.py --artifacts ml/artifacts --header firmware/include/cmoe_model.h
```

> [!WARNING]
> Synthetic data is scaffolding for smoke-testing the pipeline. **No result computed
> from it is a validation result**, and none belongs in a pitch, paper or grant
> application. Swap in your real calibration set — schema in
> [`docs/DATA_SPEC.md`](docs/DATA_SPEC.md).

### Backend

```bash
uvicorn backend.app:app --reload     # → http://127.0.0.1:8000/docs
```

### Firmware

```bash
cd firmware
pio run                 # build
pio run -t upload       # flash ESP32-S3
pio device monitor      # serial UX
```

### Tests

```bash
pytest -q        # 41 passing
ruff check .
```

## 📊 Validation status

Lab-validated on **200+ stems across 4 varieties** under controlled conditions.
Not field-validated.

| Metric | Target | Actual | Notes |
|---|---|---|---|
| Brix RMSE | ≤ 0.8 °Bx | _fill from `evaluate.py`_ | vs. benchtop refractometer |
| Conformal coverage @ α=0.10 | ≥ 90% | _fill_ | marginal **and** per-variety |
| Badge accuracy when committed | ≥ 92% | _fill_ | excludes YELLOW abstentions |
| False-GREEN rate | < 1% | _fill_ | the costly error |
| Inference latency | < 120 ms | _fill_ | ESP32-S3 @ 240 MHz, int8 |
| Backcast error (Mill Mode) | ≤ 1.2 °Bx | _fill_ | 0–48 h simulated transit |

> Fill this table from your own `evaluate.py` output. Never publish a number you
> cannot reproduce from this repository.

## 🌍 Sustainable development impact

| SDG | How Brixometer contributes |
|---|---|
| **2 — Zero Hunger** | Precise harvest timing raises sucrose recovery and cuts normalised post-harvest loss |
| **8 — Decent Work & Economic Growth** | Transit-corrected grading stops farmers absorbing losses from logistics they don't control |
| **9 — Industry, Innovation & Infrastructure** | Brings precision agriculture within reach of smallholders at 2% of lab-instrument cost |

## 👥 Who it's for

**Primary — the smallholder sugarcane farmer.** 1–5 acres, ₹1–2 Lakh per season, no lab
access, no data behind their harvest decisions. Affordable through their FPO at
₹100–500 amortised per farmer.

**Secondary — the mill intake operator.** Grades incoming lots with no way to account for
transit degradation, leaving every decision exposed to dispute. Brixometer gives them a
tamper-evident, cryptographically signed, mathematically backcasted grade.

**Extended.** FPO managers, cooperative leaders, state agriculture officers, agri-input
dealers, precision agriculture researchers.

## 🛣 Roadmap

| TRL | Stage | Status |
|:--:|---|:--:|
| 1–3 | Principles → concept → proof of concept | ✅ |
| **4** | **Validated in laboratory** | 🔵 **Current** |
| 5 | Validated in relevant environment | 🔜 Phase 1 field pilot |
| 6 | Demonstrated in relevant environment | 🔜 Phase 2 mill integration |
| 7 | System prototype demonstrated | 🔜 Phase 3 pre-production |
| 8–9 | Complete, qualified, operationally proven | 🔜 Vision 2028+ |

Full exit criteria and risk register: [`docs/ROADMAP.md`](docs/ROADMAP.md).

## ⚠️ Known limitations

Stated plainly, because a project that names its own limits is worth more than one
that overclaims. Detail in [`docs/SCIENCE.md`](docs/SCIENCE.md).

- **This predicts a refractometer reading; it does not measure Brix directly.** Valid only
  inside the range of varieties, maturities and conditions in the calibration set.
- **18 sparse channels is not a spectrum.** Expect lower accuracy than published benchtop
  cane-NIR results, and do not cite those results as if they were this device's.
- **Conformal coverage is marginal, not conditional.** Per-variety undercoverage is a
  blocking issue, and `evaluate.py` reports it separately.
- **First-order kinetics degrade past ~48 h transit**, where microbial dextran formation
  adds terms the model does not capture.
- **The Ed25519 signing routine is currently a stub.** It must be replaced with
  hardware-backed key storage before any field unit ships.

## 🤝 Contributing

Issues and PRs welcome. Run `ruff check .` and `pytest` first, and read
[`docs/CONTRIBUTING.md`](docs/CONTRIBUTING.md) — this project has a few rules that
aren't obvious:

1. Never widen a claim beyond its evidence.
2. Host/device preprocessing parity is a correctness property, not a nicety.
3. The device must be allowed to refuse — don't remove an abstention path.
4. The server stores and verifies; it never computes a grade.
5. Calibration data never touches training.

## 📄 Licence

MIT — see [`LICENSE`](LICENSE).

---

<div align="center">

Built by **Jeshwin David C** · B.Tech AI & Data Science

*Brixometer follows SSMA, and carries the same lesson forward: real problems demand real
domain knowledge, hardware and software must evolve together, and honest teamwork always
produces more reliable results than individual brilliance.*

</div>

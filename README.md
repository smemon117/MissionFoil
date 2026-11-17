# UAV Aerofoil Analysis Platform

**Mission-based aerofoil selection and first-pass UAV design, entirely in your browser.**

Choose a mission, rank aerofoils for it, inspect their polars, watch the flow in a live wind tunnel, predict new NACA sections with machine learning, and size a first-pass aircraft, all from a single HTML file. No installation, licences, solver setup or CFD expertise required.

> **Scope:** this platform is built for conceptual design and early-stage screening. It does not replace CFD or wind-tunnel testing. Verify final designs with high-fidelity analysis before construction.

---

## Contents

- [Quick start](#quick-start)
- [Modules](#modules)
- [ML model accuracy](#ml-model-accuracy)
- [How the ranking works](#how-the-ranking-works)
- [AI Predictor (Gemini)](#ai-predictor-gemini)
- [Tech stack](#tech-stack)
- [Project structure](#project-structure)
- [Limitations](#limitations)
- [Changelog](#changelog)
- [Team](#team)
- [Licence](#licence)

---

## Quick start

**Option 1: open locally**

1. Download `index.html`.
2. Open it in a modern browser (Chrome, Edge, Firefox or Safari).

An internet connection is needed the first time so that Tailwind, Plotly, Font Awesome and the Inter font can load from their CDNs.

**Option 2: GitHub Pages**

1. Push `index.html` to the root of this repository.
2. Go to **Settings → Pages**, set the source to the `main` branch, root folder.
3. The platform will be live at `https://<your-username>.github.io/<repo-name>/`.

---

## Modules

| Module | Name | What it does |
|---|---|---|
| **A** | Mission Ranker | Ranks aerofoils for a mission profile (Surveillance, Mapping, Cargo, Racing, Reconnaissance or Custom weights) at your Reynolds number. Ranks either the verified XFOIL library or the ML-predicted NACA 4-digit set. |
| **B** | Aerodynamics | Full polar analysis of the selected aerofoil: Cl, Cd and Cl/Cd against angle of attack, plus the generated aerofoil geometry. |
| **C** | Verified Library | Browse and filter the verified aerofoil library (real XFOIL polars, Re 200k–600k) by mission and aerofoil class. |
| **D** | Wind Tunnel | Real-time 2D Navier–Stokes wind tunnel with smoke, streamlines, particles, live drag/lift telemetry, adjustable angle of attack, fluid properties and custom shapes. |
| **E** | ML Predictor | Gaussian-Process surrogate that predicts full polars for any NACA 4-digit aerofoil at any Reynolds number from 200,000 to 600,000, with geometry preview and CSV import. |
| **F** | Flight Design | First-pass flight sizing: eight platform types with built-in factors (Oswald efficiency, etc.), a VTOL energy budget, battery state, polar charts and an exportable design report. |
| **AI** | AI Predictor | Looks up or estimates polars for non-NACA-4 sections (NACA 5-digit, NACA 6-series, Selig, Eppler, Clark Y and others) and can add them to the library for Modules A–C. |

---

## ML model accuracy

The Module E model is trained on **128 NACA 4-digit aerofoils (632 XFOIL polars, Re 200,000–600,000)**. It is evaluated with **5-fold cross-validation**, so every aerofoil is predicted by a model that never saw it during training.

| Output | Accuracy* |
|---|---|
| Cl max | 98.6 % |
| Cd min | 97.0 % |
| Cl/Cd max | 97.3 % |
| Stall angle | 92.2 % |
| Optimum AoA | 89.4 % |
| **Average** | **94.9 %** |

\* Accuracy = 100 % − mean absolute percentage error between predicted and true values on held-out aerofoils. Optimum AoA is the hardest output because the lift-to-drag peak is flat (average error ≈ 0.5°).

**Validity range:** NACA 4-digit sections, camber 0–8 %, thickness 6–21 %, Re 200,000–600,000. The reference polars are XFOIL results: a computational prediction, not wind-tunnel data.

---

## How the ranking works

Each eligible aerofoil gets a transparent weighted score:

```
Score = w1 · (Cl/Cd)max  +  w2 · Cl,max  +  w3 · α_stall
```

Each term is min–max normalised across the candidate pool. The default mission weights are:

| Mission | w1 (Cl/Cd) | w2 (Cl max) | w3 (stall) |
|---|---|---|---|
| Surveillance | 0.50 | 0.25 | 0.25 |
| Mapping | 0.40 | 0.30 | 0.30 |
| Cargo | 0.20 | 0.60 | 0.20 |
| Racing | 0.60 | 0.20 | 0.20 |
| Reconnaissance | 0.34 | 0.33 | 0.33 |
| Custom | user-defined | user-defined | user-defined |

Mission → aerofoil-class rules restrict each mission to the recommended aerofoil classes before scoring. Ranking runs only when you press **Rank UAV Aerofoils**.

---

## AI Predictor (Gemini)

The AI Predictor uses Google Gemini:

1. A **web-grounded request** asks for published XFOIL or wind-tunnel data (e.g. airfoiltools.com, UIUC).
2. A **plain estimate** request runs **in parallel** as a fallback.

Sourced data is preferred when it arrives within a few seconds. Every result is clearly badged:

| Badge | Meaning |
|---|---|
| **XFOIL-trained ML predictor** | NACA 4-digit input, answered by the Module E model |
| **Online sources** | Gemini found published data (links shown) |
| **AI estimate** | Gemini estimate, unverified |
| **SIMULATED** | Offline placeholder formula; never added to the library |

**API key — never commit it.** Two safe ways to enable live AI:

1. **Shared AI service (recommended for a public site):** deploy the small Cloudflare Worker in [`worker/`](worker/README.md). It keeps your Gemini key secret on the server, so every visitor can use the AI Predictor. Then set `AIS_PROXY_URL` in `index.html` to the Worker's URL.
2. **Bring your own key:** visitors paste their own Gemini key on the AI Predictor page. It is stored only in their browser's `localStorage` and sent straight to Google.

With neither, the page falls back to a clearly labelled offline placeholder.

---

## Tech stack

- **Single-file app:** HTML, CSS and vanilla JavaScript; no build step, no backend
- [Tailwind CSS](https://tailwindcss.com/) (CDN) for styling
- [Plotly.js](https://plotly.com/javascript/) 2.27 for polar charts
- [Font Awesome](https://fontawesome.com/) 6.4 for icons
- Custom 2D Navier–Stokes solver (semi-Lagrangian advection, vorticity confinement, SOR pressure projection) rendered on HTML5 Canvas
- In-browser Gaussian-Process regression for the ML predictor
- Google Gemini API for the AI Predictor (optional), via a Cloudflare Worker proxy

---

## Project structure

```
.
├── index.html     # the entire platform (all modules, data and models)
├── README.md
├── worker/
│   ├── worker.js  # Cloudflare Worker: keeps the Gemini key secret
│   └── README.md  # 10-minute setup guide
└── docs/
    └── screenshots/   # optional: add images for this README
```

To show screenshots, add them to `docs/screenshots/` and reference them here, for example:

```markdown
![Mission Ranker](docs/screenshots/module-a.png)
```

---

## Limitations

- Results are for **conceptual design and screening only**.
- The ML predictor is valid only within its training envelope (NACA 4-digit, Re 200k–600k).
- The Module D wind tunnel is a qualitative, low-resolution 2D solver. Its live Cl/Cd values illustrate trends and are not design data.
- AI Predictor results are estimates unless linked to a published source. Check the source before quoting numbers.

---

## Changelog

### v36
- British spelling throughout the interface: "aerofoil".
- Removed the embedded API key. The AI Predictor now uses an optional Cloudflare Worker proxy (key kept server-side) or the visitor's own key.

### v35
- **Mission Ranker:** ranking now starts only from the *Rank UAV Aerofoils* button, not when the mission profile or filter changes. Rapid repeat clicks run once, and changing the mission mid-ranking cancels the run.
- **Wind Tunnel:** major frame-rate fix. The solver pauses when the tunnel is not on screen; per-frame layout thrashing was removed; the canvas resolution is capped; particle drawing is batched; the pressure solve is faster and more accurate (warm-started SOR); the timestep is smoothed; adaptive quality applies on slower machines.
- **AI Predictor:** one prediction at a time (inputs lock while running, and the Enter key no longer bypasses the lock). Grounded and fallback requests run in parallel, up to ~2× faster. Repeat predictions are cached and return instantly.

---

## Team

| Role | Name | Affiliation |
|---|---|---|
| Developer | **S M Emon Rashid** | BEng Aircraft Engineering, Kingston University · smemonrashid@gmail.com |
| Supervisor | **Dr. Louie Timajo** | Dean, Faculty of Aviation Engineering, Nilai University |

---

## Licence

No licence has been chosen yet. Until one is added, all rights are reserved by the author. To let others use or build on this work, add a `LICENSE` file (for example MIT for permissive use, or CC BY-NC 4.0 to restrict commercial use).

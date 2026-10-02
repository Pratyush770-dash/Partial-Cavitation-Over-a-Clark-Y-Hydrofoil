# Partial-Cavitation-Over-a-Clark-Y-Hydrofoil
A 2D CFD study of cavitating flow over a Clark Y hydrofoil at 8° angle of attack. I lowered the cavitation number in steps (σ = 5, 2.0, 1.6) to see where vapour first appears on the suction side and how far it spreads along the chord.

I did this as a self-directed portfolio project. My interest is underwater vehicles that rely on supercavitation, and a cavity growing on a simple foil seemed like the most honest place to start learning how it works in a solver.

## Table of Contents
- [What was done](#what-was-done)
- [Software used](#software-used)
- [Reproducing the results](#reproducing-the-results)
- [Results](#results)
- [Method in short](#method-in-short)
- [Limitations](#limitations)
- [Repository structure](#repository-structure)
- [Author](#author)

## What was done

- Built a 2D Clark Y foil (chord 0.1 m) from the UIUC airfoil coordinates and cut it out of a 1.6 m × 1.0 m water domain
- Set up a mixture multiphase model in Fluent with the Schnerr-Sauer cavitation model and SST k-ω turbulence
- Ran three cavitation numbers one after another: σ ≈ 5.0 (baseline, no vapour), σ ≈ 2.0 and σ ≈ 1.6
- Set the angle of attack by tilting the inlet velocity instead of rotating the foil, so the mesh stayed simple
- Checked wall y⁺ on the foil (it stayed around 0.1 to 1.0)
- Used the pressure coefficient plot to find where cavitation should start, then confirmed it with the vapour volume and the Cp plateau at σ = 1.6
- Got two things wrong early on and fixed them: the first run diverged, and the first stable run gave negative lift because I had the sign of the inlet Y-velocity wrong (details in the report)

## Software used

- ANSYS DesignModeler for the geometry
- ANSYS Meshing for the mesh
- ANSYS Fluent 2026 R1 (Student), 2D, double precision
- Fluent's own XY-plot, contour and report tools for post-processing

## Reproducing the results

1. Open Workbench and add a Fluid Flow (Fluent) system. Right-click Geometry, go to Properties and set Analysis Type to 2D.
2. Get `clarky.dat` from the UIUC airfoil database. Multiply x and y by 0.1, set z = 0, and save the points as `group point x y z` rows (upper surface as group 1, lower as group 2, and a short line across the trailing edge as group 3). The file I used is in `geometry/`.
3. In DesignModeler: Concept → 3D Curve → From Coordinates File. Then Concept → Surfaces From Edges for the foil. Draw the rectangle from x = −0.5 to 1.1 m and y = −0.5 to 0.5 m, and use Create → Boolean (Subtract) to cut the foil out. Delete the leftover line bodies so the tree shows 1 part, 1 body.
4. In Meshing, create named selections: `inlet` (left), `outlet` (right), `top`, `bottom`, and `foil` (all three foil edges).
5. Mesh controls: edge sizing on `foil`, an inflation layer on the foil with a first layer of 2×10⁻⁶ m, 40 layers and growth rate 1.15, and a sphere of influence (radius 0.18 m, centred at x = 0.15 m) to refine the foil and wake region. Use the triangle method.
6. In Fluent: pressure-based, 2D planar, mixture model with two phases (water-liquid primary, water-vapour secondary), SST k-ω, operating pressure 0 Pa.
7. Cavitation: mass transfer from liquid to vapour, Schnerr-Sauer, vaporization pressure 3169 Pa, bubble number density 1×10¹³.
8. Boundaries: `inlet`, `top` and `bottom` are velocity inlets with components (9.903, +1.392) m/s. `outlet` is a pressure outlet. `foil` is a no-slip wall.
9. Set the outlet pressure for the σ you want (table below), and put the same value in Reference Values → Pressure.
10. Run the cases in order without re-initialising, starting from the baseline. Check the inlet pressure with a surface integral afterwards to confirm the real σ.

## Results

Cavitation number is σ = (p∞ − p_v) / (½ρU²). With U = 10 m/s and ρ = 997 kg/m³, ½ρU² = 49,850 Pa, so the outlet pressure is p_v + σ × 49,850.

| Target σ | Outlet pressure (Pa) | Inlet pressure measured (Pa) | σ actually reached |
|---|---|---|---|
| 5.0 | 252,419 | 251,867 | ≈ 5.0 |
| 2.0 | 102,869 | 102,505 | ≈ 1.99 |
| 1.6 | 82,929 | 82,548 | ≈ 1.59 |

| σ | Cl | Cd | What the cavity looks like |
|---|---|---|---|
| 5.0 | about 1.06 (not fully settled) | about 0.054 | No vapour |
| 2.0 | about 0.91 | about 0.061 | Vapour starts to form, very small |
| 1.6 | about 0.92 | about 0.060 | Thin sheet at the leading edge, about 0.1 of the chord |

Cl and Cd are per metre of depth, divided by ½ρU²c = 4,985 N/m, and I read them off the plots by eye.

The baseline Cp plot has a suction-side minimum of about −1.9 near the leading edge. That is why σ = 3 and σ = 2.0 give almost nothing, and why vapour shows up once σ drops below about 1.9. At σ = 1.6, the Cp curve goes flat at roughly −1.6 from the leading edge to about x = 0.01 m, which is the signature of a vapour pocket sitting on the surface.

Plots are in `results/figures/`: Cp at σ = 5 and σ = 1.6, wall y⁺, lift, drag and vapour volume histories, and the residuals.

## Method in short

- **Geometry:** Clark Y at 0° in the mesh. The 8° angle of attack comes from the inlet velocity direction, which means the top and bottom boundaries are also velocity inlets.
- **Mesh:** triangles with a boundary-layer inflation on the foil (first layer 2×10⁻⁶ m for y⁺ near 1) and a sphere of influence around the foil and wake.
- **Physics:** Re ≈ 1.1×10⁶ based on chord, water at 25 °C (ρ = 997 kg/m³, μ = 8.9×10⁻⁴ Pa·s, p_v = 3169 Pa).
- **Stepping σ:** change the outlet pressure and the reference pressure, then keep iterating from the previous solution. The three cases ran at roughly iterations 0–1000 (σ = 5), 1000–2500 (σ = 2.0) and 2500–4000 (σ = 1.6).
- **Checking σ:** an area-weighted average of static pressure on the inlet, because the pressure level around the foil is not exactly the number typed at the outlet.

## Limitations

- **No mesh independence study.** I planned three meshes (about 50k, 100k and 200k cells) but only ran one. The results should be read as qualitative.
- **No validation against an experiment.** I did not compare the cavity length against a published Clark Y cavitation tunnel paper. The only checks I did are internal ones (Cp ≈ −σ under the cavity, y⁺, convergence).
- **Only three cavitation numbers.** σ = 1.3, 1.0 and 0.8 were planned but not run because of time. So there is no lift breakdown here and no cloud shedding. The cavity is at the inception or short-sheet stage.
- **The cavity had not stopped growing.** The vapour volume was still rising at the end of the σ = 1.6 run. Lift and drag looked flat, but the cavity itself was not fully settled.
- **Baseline lift was still drifting** when I moved on from σ = 5, so comparing it with σ = 2.0 and 1.6 would overstate any cavitation effect.
- **Drag looks high.** Cd of about 0.055 to 0.06 is more than I'd expect for a Clark Y at this Reynolds number. I haven't cross-checked it against XFOIL. The blunt trailing edge could be part of it, but I haven't tested that.
- **No zoomed vapour contour.** Fluent froze when I tried to save one for σ = 1.6, and the full-domain contours are too zoomed out to show the cavity.
- **Bubble number density was left at the default** (1×10¹³) and not tuned.

## Repository structure

```
├── README.md
├── report/
│   └── Cavitation_Hydrofoil_Report.docx
├── geometry/
│   └── clarky_data.txt        → coordinate file for DesignModeler
├── mesh/
│   └── CFD_mesh
├── case_files/
│   └── cavitation_simulation
└── results/
    └── figures/                   → Cp, y+, lift, drag, vapour volume, residuals
```

## Author

**Pratyush Dash**

B.Tech Chemical Engineering, KIIT University, Bhubaneswar

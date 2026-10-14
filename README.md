<p align="center">
  <img src="media/logo.svg" alt="OptoNorm Logo" width="280" />
</p>

# OptoNorm: High-Precision Optometric Clinical Shorthand Normalizer & FHIR R4 Exporter

[![CI](https://img.shields.io/badge/CI-passing-brightgreen.svg)](https://github.com/ctasca/optonorm/actions)
[![Version: 0.48.2](https://img.shields.io/badge/version-0.48.2-blue.svg)]()
[![Python 3.13](https://img.shields.io/badge/python-3.13+-blue.svg)](https://www.python.org/)
[![Frontend: React 19](https://img.shields.io/badge/frontend-React%2019-61dafb.svg)]()
[![Tests](https://img.shields.io/badge/tests-4231%20passed-brightgreen.svg)]()
[![Benchmark](https://img.shields.io/badge/gold%20benchmark-100%25-success.svg)]()

[![Code style: ruff](https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/astral-sh/ruff/main/assets/badge/v2.json)](https://github.com/astral-sh/ruff)
[![Latency](https://img.shields.io/badge/mean%20latency-1.81%20ms-orange.svg)]()
[![Hallucinations](https://img.shields.io/badge/hallucinations-0.00%25-red.svg)]()
[![License: BSL 1.1](https://img.shields.io/badge/License-BSL_1.1-blue.svg)](LICENSE.md)

**OptoNorm** is a specialized clinical NLP engine engineered for Optometry and Ophthalmology electronic health record (EHR) systems. It bridges the gap between raw, conversational Speech-to-Text (ASR) transcripts and standardized clinical notation.

Instead of outputting verbose, generic prose (e.g., _"the patient's visual acuity was measured at twenty out of twenty in the right eye"_), OptoNorm deterministically extracts and replaces clinical measurement spans with concise, industry-standard clinical shorthand (`VA OD 20/20`), while leaving ambient clinician narrative **100% verbatim**. Measurement findings map to **HL7 FHIR R4 Observation** resources (LOINC / SNOMED-CT). Spoken ocular medications map to **MedicationStatement** resources (RxNorm). The normalized note, structured findings, and alerts are a draft for the optometrist to verify.

---

## Key Capabilities & Design Guarantees

1. **Deterministic Typed Grammars**:
   - **Visual Acuity**:
     - **Snellen & Metric**: Distance Snellen (Imperial `20/10`–`20/800` including intermediate lines `20/125`, `20/12`, Metric `6/3`–`6/120` including `6/3.8`, `6/4`, `6/5`, `6/7.5`, `6/12`, `6/60`), pinhole (`PH 20/25`, `PH 6/7.5`), monocular (`OD`, `OS`) and binocular (`OU`) laterality. Spoken _"20 to 25"_, _"20 slash 25"_, _"20 forward slash 25"_, and _"6 to 12"_ are fractions. _"Cup to disc"_ is not.
     - **LogMAR & ETDRS Letters**: Decimal logMAR bounded strictly at $\le 1.30$ (`logMAR 0.1`, `logMAR -0.2`, `logMAR 0.0`) mapped to LOINC `LL3783-9` and ETDRS letter scores (`ETDRS 85 letters`, `85 letters read`) mapped to LOINC `55282-8`.
     - **Low-Vision & Pediatric Behaviors**: Qualitative low-vision projection (`LPP`, `LP c proj`, `LP s proj`, `CF @ 3ft`, `HM`, `LP`, `NLP`) and pediatric behavioral assessments (`F&F` Fix and Follow, `CSM` Central Steady Maintained across 5 languages, `BTL` Blinks to Light across 5 languages).
     - **Near Acuity & Typography Scales**: French Parinaud (`P1.5`–`P14`), Italian/Spanish Jaeger (`J1`–`J7`), Austrian/German Nieden (`N1`–`N8`), Birkhäuser (`B1`–`B6`), and British typographical N-point print (`N5`–`N48`). Disambiguates shared tokens (`N5`, `N6`, `N8`) between Nieden and British N-point using typographical cues (`point`, `print`, `British`).
   - **Visual Acuity Tails & Outcome Clauses**: Recognizes visual acuity outcome tails following manifest refraction refinement (e.g. _"giving 2020"_ $\rightarrow$ `giving VA 20/20`, _"Also 2020"_ $\rightarrow$ `VA 20/20`), and binocular balance endpoints (_"Nuclear balance accepted with both eyes open. 2020"_ $\rightarrow$ `Binocular balance: 20/20 OU`).
   - **Explicit Distance & Near Differentiation**:
     - Explicit _"distance visual acuity"_ / _"distance acuity"_ $\rightarrow$ **`DVA`** (e.g., `DVA OD 20/400 sc`, `DVA OD 6/12 PH 6/7.5 sc`, `DVA OD logMAR 0.1`). Contextually propagates across contralateral eyes in the same distance test sequence.
     - Explicit _"near visual acuity"_ / _"near vision"_ $\rightarrow$ **`NVA`** (e.g., `NVA OU 20/20`, `NVA OD N5`).
     - General _"visual acuity"_ $\rightarrow$ canonical **`VA`** (e.g., `VA OD 20/20`).
   - **Correction State Extraction**: Standardizes unaided (_sine correctione_ $\rightarrow$ **`sc`**) and aided (_cum correctione_ $\rightarrow$ **`cc`**) conditions (e.g., `DVA OD 20/400 sc`).
   - **Subjective & Objective Refraction**:
     - Standard dictation: Sphere, cylinder, axis, and reading add powers (`OD -2.50 -0.75 x 180 Add +2.00`).
     - **Advanced Refractive Diagnostics, Cycloplegia, Retinoscopy & Lensometry (Phase 1B)**:
       - **Manifest Diagnostics**: Spherical equivalent (`SE -3.25 D`, LOINC `70948-5`), back vertex distance (`BVD 12 mm` / `HSA 14 mm`, LOINC `95289-5`), intermediate add (`Int Add +1.25 D`, LOINC `95382-8`), and monocular pupillary distance (`Mono PD 31.5/32.5`). Multilingual spoken support across EN, FR, DE, IT, ES.
       - **Autorefraction**: Reliability / confidence scores (`Conf 8`, `*`), pupil diameter (`PS 4.5 mm`), companion autokeratometry (`ARK`), and `RefractionStage.AUTOREFRACTOR_DRY` / `AUTOREFRACTOR_WET`.
       - **Cycloplegic Refraction**: Agent specification (`Cyclo 1%`, `Tropicamide`, `Atropine`), post-drops elapsed time (`30 min post-drops`), residual accommodation (`res acc 0.50 D`), tonus deduction (`Tonus -0.75 D`), and `RefractionStage.CYCLOPLEGIC`.
       - **Retinoscopy**: Gross vs. Net distinction (`Gross Ret:`, `Net Ret:`), working distance deduction (`WD -1.50 D`), reflex motion (`with`, `against`, `neutrality`), dynamic retinoscopic lag (`MEM +0.50 D`), and `RefractionStage.RETINOSCOPY`.
       - **Lensometry & Habitual Correction**: Multifocal lens designs (`SV`, `FT-28`, `FT-35`, `PAL`, `Prog`), segment height (`Seg ht 18 mm`, SNOMED CT `419309008`), optical center height (`OC ht 22 mm`), slab-off bicentric grinding prism (`slab-off 2Δ BD`), and `RefractionStage.HABITUAL`.
     - Isolated cylinder clauses with dysfluency resilience (`"-0.75 cylinder Cylinder at 85° for the left eye"` $\rightarrow$ `OS -0.75 x 085`, `"-1.75 sphere -0.50 cylinder cylinder at 20°"` $\rightarrow$ `OD -1.75 -0.50 x 020`) and standalone reading add prescriptions (`Add +1.75 OU`, `"Frenier, the AD is plus 2.00 diopters in both eyes"` $\rightarrow$ `Add +2.00 OU`).
     - **3-to-5 Number Rapid Sequence Dictation**: Clinicians dictating rapid numeric sequences without parameter words, with or without spoken connectors like _"and"_ / _"at"_ / _"x"_ (e.g., _"Manifest Refraction is 1 1 40"_ $\rightarrow$ `+1.00 -1.00 x 040`, _"Manifest Refraction is 1 1 and 40"_ $\rightarrow$ `+1.00 -1.00 x 040`, _"Refraction is 1 1 and 1"_ $\rightarrow$ `+1.00 -1.00 x 001`, _"Left eye is -1 2 40"_ $\rightarrow$ `OS -1.00 -2.00 x 040`, _"Right eye is -2.50 -0.75 180 2.00 64"_ $\rightarrow$ `OD -2.50 -0.75 x 180 Add +2.00 PD 64`). Unsigned cylinder still follows the minus-cylinder convention. An unsigned sphere is proposed as plus and flagged for review, because that sign was not spoken. A hundredths cylinder is the exception: _"1 to 15 and 40"_, _"1 too 15 and 40"_, and _"1 two 15 and 40"_ are the same as _"1 2 15 and 40"_ and are written as `+1.00 -2.15 x 040`. _"1 1 and 40"_ stays a reviewed plus proposal.
   - **Contact Lens Examination, Brands & Over-Refraction**:
     - Standardizes contact lens specifications (Brand, Modality, Base Curve, Diameter, Sphere, Cylinder, Axis, Add Power) e.g., `Biofinity OD: Toric / BC 8.7 / Dia 14.5 / -2.00 -1.25 x 180`.
     - Contact lens over-refraction (`CL OR OD: plano -0.50 x 010 -> 20/20`, `CL OR OS: +0.25 -> 20/20`).
     - **Intelligent Brand & Acoustic Matcher (`contact_lens_catalog.py`)**: Master catalog of 35+ commercial contact lenses across Johnson & Johnson, Alcon, CooperVision, Bausch + Lomb, and Specialty labs (RGP, Scleral, Ortho-K, Hybrid) with FDA material group, Dk/t, water content, base curves, diameters, and modalities. Resolves severe ASR acoustic phonetic slips (*"daily's total won toric"* $\rightarrow$ `Dailies Total 1 for Astigmatism`, *"bio affinity toric"* $\rightarrow$ `Biofinity Toric`, *"aqua view oasis"* $\rightarrow$ `Acuvue Oasys`, *"clarity one day"* $\rightarrow$ `Clariti 1 Day`, *"precision won"* $\rightarrow$ `Precision1`, *"total thirty toric"* $\rightarrow$ `Total 30 for Astigmatism`) while enforcing zero false positives on negative controls.
   - **Prism Diopter & Binocular Alignment**: Parses horizontal (`BI`, `BO`) and vertical (`BU`, `BD`) prism powers with the prism delta symbol (`Δ`), supporting monocular prescriptions (`2Δ BO OD`, `1.5Δ BU OD`) and compound horizontal/vertical prisms (`2Δ BO 1.5Δ BU`).
   - **Corneal Keratometry (K-Readings) & Curvature Radius (Phase 2)**:
     - Standardizes manual and automated keratometry diopters and principal meridians into flat K, steep K, and corneal astigmatism cylinder (`K OD: 43.00 @ 180 / 44.25 @ 090`, `K OS: 42.75 @ 005 / 44.50 @ 095`).
     - Standardizes radius of curvature in millimeters (`r1 7.85 mm, r2 7.60 mm`), delta K (`ΔK 1.25 D WTR`), WTR/ATR astigmatism orientation, and mire image quality (`clear mires`, `distorted mires`).
   - **Pachymetry, Corneal Thickness & DEWS II Dry Eye Diagnostics (Phase 2)**:
     - Ultrasound & Optical Central Corneal Thickness (CCT): Standardizes micrometer measurements (`CCT: 515 OD, 520 OS µm`, `CCT: 540 OU µm`).
     - Thinnest Corneal Thickness (`TCT: 510 OD, 515 OS µm`, local code `thinnest-corneal-thickness`), Epithelial Thickness Mapping (`Epi map: 52 OD, 50 OS µm`), and measurement modality capture (`USP`, `OCT`, `Pentacam`). Central corneal thickness stays on LOINC `71813-0`.
     - Fluorescein Tear Film Breakup Time: Breakup times in seconds (`TBUT: 4s OD, 3s OS`, `TBUT: 5s OU`, LOINC `71811-4`) and Schirmer test (`Schirmer: 12 mm OD, 10 mm OS`, LOINC `70957-6`).
     - TFOS DEWS II Diagnostic Battery: Non-Invasive TBUT (`NIBUT: 6s OD, 7s OS`, local code `nibut`), Tear Meniscus Height (`TMH: 0.15mm OD, 0.20mm OS`), Tear Osmolarity (`Osmolarity: 315 OD, 308 OS mOsm/L`, LOINC `85382-0`), and qualitative MMP-9 InflammaDry immunoassay (`InflammaDry (+) OD, (-) OS`, LOINC `107455-8`).
   - **Tonometry (IOP) & Advanced Diagnostic Modalities**:
     - Standardizes Goldmann Applanation Tonometry (`GAT`), Perkins (`PAT`), Pascal Dynamic Contour Tonometry (`DCT`), Pneumotonometry (`Pneumo`), Schiøtz, Tono-Pen (`Tonopen`), iCare, and Non-Contact Air Puff (`NCT`) (`IOP: 14 OD, 15 OS mmHg`, `IOP (PAT): 15 OD, 16 OS mmHg`, `IOP (DCT): OD 18 mmHg`, `IOP (Pneumo): 16 OU mmHg`, `IOP (Schiotz): OD 17 mmHg`).
     - Extracts patient posture (`sitting`, `supine`) e.g., `IOP (PAT, supine): 14 OD, 15 OS mmHg`.
     - Extracts Tono-Pen reliability and confidence ratings (`5%`, `10%`, `>95%`, `<5%`) e.g., `IOP (Tono-Pen 5%): 15 OU mmHg`.
     - Parses Ocular Response Analyzer (`ORA`) and Pascal DCT biomechanical parameters: Goldmann-correlated IOP (`IOPg`), Corneal-compensated IOP (`IOPcc`, LOINC `96093-0` OD / `96092-2` OS), Corneal Hysteresis (`CH`), and Ocular Pulse Amplitude (`OPA`). Generates discrete monocular HL7 FHIR R4 Observations with discrete components and clinical notes under LOINC `55284-4` (_Intraocular pressure_).
   - **Dynamic Cup-to-Disc (C/D) Ratio**: Scalar symmetric (`C/D 0.3 OU`), monocular asymmetric (`C/D: 0.3 OD, 0.7 OS`), biaxial horizontal/vertical ratios (`C/D OD 0.4H/0.45V`, `C/D OS 0.5H/0.7V (inf notch)`), and compact meridian pairs in either order (`C/D 0.7V/0.5H`). Neuroretinal rim notching (`inf notch`, `sup notch`), ISNT rule status (`ISNT intact`, `ISNT violated`), Drance disc hemorrhage, and beta-zone peripapillary atrophy (`PPA beta`) are captured across all 5 languages. The glaucoma-suspect flag uses the same $\ge 0.65$ cutoff as CDS, applied to a scalar ratio or to the vertical meridian when one was parsed. Horizontal enlargement alone does not set the flag. LOINC `70949-3`.
   - **Contact Lens Fit & Corneal Topography (Phase 6)**: Multifocal adds `Low` / `Med` / `High`. Toric rotation in degrees `CW` or `CCW`, including `LARS`. Soft-lens push-up `0–100%` and movement on blink `0–3 mm`. Scleral apical vault `0–1000 µm`, limbal clearance, and landing-zone blanching. RGP fluorescein: apical clearance, apical bearing, edge lift, and `3 and 9 o'clock staining`. Spherical over-refraction (`SOR +0.50`, −20 to +20 D) and spherocylindrical over-refraction (`SCOR pl/-0.75x180`). Simulated keratometry keeps flat and steep meridians with axes (`Sim K 43.50 @ 180 / 45.25 @ 090`, 30–70 D). `Kmax` is 30–90 D. `I-S` is −10 to +10 and `BAD-D` is −1 to 20. Cone morphology is nipple, oval, globus, or `PMD`. A bare keratometry pair stays keratometry. Fit measurements and `SOR` / `SCOR` use `urn:optonorm:fhir:CodeSystem:contact-lens-fit`. Topography is SNOMED `252830007` and CPT `92025`, with indices on `urn:optonorm:fhir:CodeSystem:corneal-topography`.
   - **Binocular Vision, Motility & Confrontation (Phase 5)**: Intact motility (`EOMI`, `FROM`) and cardinal-gaze grades from −4 to −1 and +1 to +4 (`−2 IO OD`). Cranial-nerve palsy (`CN III`, `CN IV` with Bielschowsky, `CN VI` abduction deficit) and jerk or pendular nystagmus. Cover-test additions: Hirschberg, Krimsky prism, A- and V-patterns, `DVD`, `DHD`, incyclophoria, excyclophoria, and Double Maddox Rod cyclotropia. Duochrome (`R/G equal`, red or green clearer), von Graefe balance, and near point of convergence to the nose, with accommodative or penlight targets and convergence insufficiency kept apart from `CI-DME`. Pupil additions: neutral-density RAPD in log units (`RAPD 0.6 log`, 0.3–3.0), reverse RAPD, Marcus Gunn pupil, brisk/sluggish/fixed light reflex, light-near dissociation, and peaked, irregular, or corectopic pupils. Stereopsis in arcseconds (`40"`, fly only, nil stereo) for Titmus, Randot, TNO, and Lang I/II, plus Worth 4-dot fusion, diplopia, and suppression. Confrontation fields: `FTFC`, intact quadrants `ST` / `SN` / `IT` / `IN`, and hemianopia, quadrantanopia, altitudinal defect, or central scotoma. FHIR uses LOINC `79803-3` for ocular motility, SNOMED `252853008` for stereotests, SNOMED `415850001` for a Worth 4-dot-only result, and SNOMED `410560002` for confrontation fields. Grades and arcseconds use `urn:optonorm:fhir:CodeSystem:binocular-vision`.
   - **Gonioscopy, Humphrey Fields & OCT (Phase 4)**: Shaffer grades 0–4, including four quadrants, Spaeth compound shorthand with iris insertion A–E, 5° angle steps from 5° through 45°, configuration `r` / `s` / `q` / `p` (plateau), and pigment 0–4+ (`D40r 2+`, `B15p 3+`). Scheie I–IV, angle structures `SL` / `TM` / `SS` / `CBB`, appositional versus synechial closure, PAS clock hours, and angle neovascularization. HVF `24-2` / `30-2` / `10-2` with SITA Faster matched before SITA Fast before SITA Standard, plus `MD`, `PSD`, `VFI`, GHT (`WNL`, borderline, `GHT ONL`, general reduction), and reliability (`FL 1/14`, `FP 2%`, `FN 5%`). Macular OCT (`CST`, cube volume, `IRF`, `SRF`, pigment epithelial detachment, `EZ intact` / `EZ disrupted`) and optic-nerve OCT (`RNFL`, `TSNIT`, `GCC`, `GCL-IPL`), with signal `8/10` or `Q > 20 dB`. `GHT ONL` is a perimetry category stored as `outside_normal_limits`. OCT normative status is only `within_normal_limits`, `borderline`, or `outside_normal_limits`, and OCT `ONL` is the outer nuclear layer. FHIR uses SNOMED `76949005` for gonioscopy and `103752008` for perimetry. Field indices and OCT measurements use `urn:optonorm:fhir:CodeSystem:glaucoma-diagnostics`. Retinal OCT is CPT `92134` and optic-nerve OCT is CPT `92133`.
   - **ICDR Diabetic Retinopathy, Posterior Lesions & Ophthalmoscopy (Phase 3)**: No apparent DR (`No DR`, `NDR`), mild through very severe NPDR including the 4-2-1 rule, and high-risk versus non-high-risk PDR. Lesion tokens `IRMA`, `VB`, `NVD`, `NVE`, `PRH`, and `VH` (vitreous hemorrhage, not a Van Herick grade). Center-involved DME (`CI-DME`) is kept apart from convergence insufficiency. AREDS drusen sizes (small $<63$ µm, intermediate $63$–$125$ µm, large $>125$ µm, `RPD`), dry versus wet AMD, `GA`, `CNV`, `ERM`, `FTMH`, `CSCR`, horseshoe tear, operculated hole, retinoschisis, pavingstone, `WWOP`, macula-on/off `RRD`, and `CRAO` / `BRAO` / `CRVO` / `BRVO`. Ophthalmoscopy modality: `BIO c depression`, `BIO s depression`, `90D exam`, `78D exam`, `Goldmann 3-mirror`. FHIR uses SNOMED `53524009` for ophthalmoscopy and the ICDR severity concepts (`312903003` mild NPDR through `312907002` high-risk PDR, `721103006` when no DR is seen, `871778008` / `871781003` for center-involved and non-center-involved DME).
   - **Pupillary Examination & RAPD**: Standard findings (`PERRLA`, `PERRL (-) RAPD`), measured dynamic light reflex diameters / anisocoria (`Pupils: 4->2mm OD, 3->2mm OS`, `Pupils: 4->2mm OU`), static diameters (`Pupils: 5mm OD, 3mm OS`), and graded Relative Afferent Pupillary Defects (`2+ RAPD OS`, `1+ RAPD OD`, `trace RAPD OS`). LOINC `80315-5`, `80313-0`, and `76504-0`.
   - **Slit Lamp Severity, Biomicroscopy Grading & Anterior Chamber (Phase 2)**:
     - **LOCS III Continuous Decimal Grading**: Continuous decimal grading for Nuclear Opalescence (`NO 0.1`–`6.9`), Nuclear Color (`NC 0.1`–`6.9`), Cortical (`C 0.1`–`5.9`), and Posterior Subcapsular (`P 0.1`–`5.9`) cataract opacities (`Lens: LOCS III: NO 3.2, NC 3.5, C 2.0, P 1.5 OD`, local codes `nuclear-opalescence`, `nuclear-color`, `cortical`, `posterior-subcapsular`), alongside conventional integer plus grading (`Lens: 2+ NS, 1+ PSC OU`).
     - **SUN Standardization for Anterior Chamber & Vitreous**: Standardizes anterior chamber cells including `0.5+` cells (1–5 cells per 1mm slit beam), qualitative flare terms (`faint` $\to$ 1+, `moderate` $\to$ 2+, `marked` $\to$ 3+, `intense/fibrin` $\to$ 4+), and vitreous haze (0 to 4+) (`AC: 0.5+ cells, faint flare OU`, `Vitreous haze: 2+ OD`, LOINC `70950-1`).
     - **Anterior Chamber Depth (Van Herick 0–4)**: Standardizes peripheral anterior chamber depth across Van Herick grades 0 through 4 (`Van Herick grade 2 OD`, `Van Herick 4 OU`, SNOMED CT `363948008`). Millimeter anterior-chamber depth and bare `VH` (vitreous hemorrhage) are not Van Herick grades.
     - **Ocular Surface Staining**: Oxford grading scale (grades 0–V) and NEI grid score (0–15) for fluorescein and lissamine green corneal/conjunctival staining (`Cornea: Oxford grade II OD`, `NEI grid score: 4 OD`).
     - **Slit Lamp Adnexa, Cornea, Iris & Pseudophakia**:
       - Meibomian Gland Dysfunction grading (`MGD 2+ OU`, `MGD: 2+ OD, 1+ OS`).
       - Conjunctival injection (0–4+), ciliary flush, and subconjunctival hemorrhage (`Conj: SCH OD`, `Conj: 2+ injection OU`, `ciliary flush OD`).
       - Corneal infiltrates with dimensional measurements (`Corneal infiltrate: 1.5x2.0 mm OD`), punctate epithelial erosions (`PEE OU`) and persistent epithelial defects (`PED OD`), keratic precipitates (`KPs: mutton-fat OD`, `fine KPs OS`), and cornea guttae (`Cornea guttae present OU`). Negated guttae (`no guttae`) stay absent. Ungraded PCO stays `present`. A YAG status is recorded only when the note says open, patent, or clear.
       - Pseudophakic IOL status and centration (`PCIOL well-centered OD`, `ACIOL OS`, SNOMED CT `261665006`), posterior capsular opacification (`PCO 2+ OS`, SNOMED CT `415252003`), and YAG laser capsulotomy status (`patent YAG capsulotomy OS`).
   - **Strabismus & Binocular Alignment**: Cover test distance and near phorias (`CT: dist 4Δ EP, near 8Δ EP`, `CT: dist 4Δ XP, near 10Δ XP`), monocular and alternating tropias (`15Δ LXT`, `20Δ RET`, `10Δ XT`), and orthophoria (`Ortho dist & near`, `Ortho dist`, `Ortho near`, `Ortho`). Discrete FHIR observation with LOINC `70951-9` (*Ocular alignment*).
   - **Clinical Grading & Modifiers**: Standardizes clinical severity grading (`one plus` $\rightarrow$ `1+`, `two plus` $\rightarrow$ `2+`, `three plus` $\rightarrow$ `3+`, `four plus` $\rightarrow$ `4+`, `grade one` $\rightarrow$ `grade 1`).
   - **Fixed-Phrase Lexicon**: Maps colloquial findings to canonical acronyms (`PERRLA`, `PERRL (-) RAPD`, `EOMI`, `DFE`, `DFE: normal`, `C/D 0.3 OU`, `AC: D&Q OU, no c/f`, `Lens: trace NS OU`, `Cornea: clear OU`, `Macula: flat, dry OU`, `Retina: flat, intact OU`, `Vitreous: clear OU`, `Conjunctiva: quiet OU`, `Iris: normal pattern OU`, `Lids/Lashes: clear OU`).

2. **Zero-Hallucination Guarantee (Post-Hoc Mathematical Entailment Guard)**:
   - Evaluates every generated shorthand token against the source speech span.
   - Multilingual entailment routing dynamically queries the active `LocaleProvider.get_source_number_inventory()` to account for locale-specific written digits, compound number words, spoken fractions, and decimal commas across English, French, Italian, Spanish, and German.
   - If even a single digit cannot be mathematically derived from the source tokens, the edit is flagged or aborted. Hallucination rate: **0.00%**.

3. **Physiological & Clinical Boundary Validation**:
   - Validates diopter increments (must be in 0.25 D steps), cylinder sign conventions, and astigmatic axes ($1^\circ$ to $180^\circ$). A non-zero cylinder requires an axis even when a sphere is already present; a zero cylinder is a spherical lens and needs no axis. An axis dictated on the next turn still completes that cylinder (`"-3 - 1."` then `"120."`). Also validates physiological intraocular pressure ranges (4–70 mmHg), valid Snellen denominators (including metric 6m denominators $4, 5, 6, 7.5, 9, 12, 15, 60$ and low-vision $300$ and $400$), central corneal thickness (300–850 µm), TBUT (1–60 s), optic nerve cup-to-disc ratios (0.0–1.0), and pupillary diameters (1.0–9.0 mm).
   - Locale-specific acuity boundary guards enforce physiological limits for Monoyer decimal scale (`1/20`, `1/10` to `10/10`), Parinaud French near acuity (`P1.5` to `P14`), Jaeger Italian/Spanish near acuity (`J1` to `J7`, with `+`/`-` modifiers), and German DIN 58220 decimal Visus (`1,0` to `0,05`), Nieden (`N1` to `N8`), and Birkhäuser (`B1` to `B6`) near reading scales.

4. **Speech-to-Text (ASR) Acoustic Artifact Resilience & Speech Dysfluency Preprocessor**:
   - **Self-Correction Engine (`self_correction.py`)**: Detects spoken reset and retraction operators across 5 languages (English: *"no wait"*, *"sorry"*, *"make that"*, *"scratch that"*, *"correction"*; French: *"pardon"*, *"non attendez"*, *"je voulais dire"*; German: *"Entschuldigung"*, *"nein warte"*, *"Korrektur"*, *"besser gesagt"*; Italian: *"scusa"*, *"no aspetta"*, *"volevo dire"*; Spanish: *"perdón"*, *"no espera"*, *"quise decir"*). When a clinician retracts a finding mid-utterance (e.g. `[VA] + "no wait" + [VA]`), OptoNorm supersedes the earlier span, marking it as discarded speech while preserving the intended clinical measurement.
   - **ASR Number-Punctuation Re-Stitcher (`punctuation_repair.py`)**: Repairs punctuation fractures inserted by acoustic pause detection in commercial ASR models (e.g., `"axis 1. 80"` $\rightarrow$ `"axis 180"`, `"minus 2. 50"` $\rightarrow$ `"minus 2.50"`, `"14. mmHg"` $\rightarrow$ `"14 mmHg"`).
   - **Stutter & Repetition Collapser (`repetition.py`)**: Collapses immediately duplicated clinical keywords, signs, laterality, and modalities (e.g., `"minus minus"`, `"sphere sphere"`, `"cylinder cylinder"`, `"right eye right eye"`).
   - **Phonetic & Acoustic Artifact Repairs (`acoustic_repair.py`)**:
     - Phonetic decimals: `"minus OH .25 cylinder"` $\rightarrow$ `-0.25 cylinder`
     - Concatenated 4-digit Snellen numbers: `"2020"` $\rightarrow$ `20/20`, `"2400"` $\rightarrow$ `20/400`, `"2300"` $\rightarrow$ `20/300`, `"2200"` $\rightarrow$ `20/200`, `"2100"` $\rightarrow$ `20/100`, `"2080"` $\rightarrow$ `20/80`
     - Prepositional laterality expansion: Normalizes phrases like `"for the right eye"` $\rightarrow$ `OD`, `"for the left eye"` $\rightarrow$ `OS`, `"for both eyes"` $\rightarrow$ `OU`
     - Hybrid compound spoken numbers: ASR digit-word hybrids like `"at 100 sixty-eight degrees"` or `"100 68 degrees"` $\rightarrow$ `axis 168`
     - Spoken degree symbols: `"at 175°"` $\rightarrow$ `axis 175`, `"at 10°"` $\rightarrow$ `axis 010`
     - Phonetic acoustic near-add variants: Conversational phrasing (`"frenir"`, `"frenier"`, `"for near"`)
     - Spoken pressure units & word numbers: `"15 mm of mercury"` $\rightarrow$ `15 mmHg`, `"twenty-one millimeters of mercury"` $\rightarrow$ `21 mmHg`
     - Distance-acuity mishearings beside an acuity word: English `"distillation acuity"` $\rightarrow$ `distance acuity`; French `"de lin"` $\rightarrow$ `de loin`; Italian `"lontanno"` $\rightarrow$ `lontano`; Spanish `"de lexos"` $\rightarrow$ `de lejos`; German `"Fernwiesus"` $\rightarrow$ `Fernvisus`
   - **Decimal-Less Diopter & Refraction Normalizer (`refraction_repair.py`)**:
     - Rapid sequence integer diopters: Normalizes unpunctuated 3-to-5 number sequences (`"-1 2 40"` $\rightarrow$ `-1.00 -2.00 x 040`, `"1 1 and 40"` $\rightarrow$ `+1.00 -1.00 x 040`, `"1 1 and 1"` $\rightarrow$ `+1.00 -1.00 x 001`, `"125 075 8"` $\rightarrow$ `-1.25 -0.75 x 008`)
     - Compound cylinder-axis concatenation: Unpacks conjoined tokens (e.g. `"-0.75x180"` $\rightarrow$ `-0.75 x 180`, `"-75120"` $\rightarrow$ `-0.75 x 120`)
     - Habitual glasses measurement dictation: Standardizes phrases like `"Glasses measured minus 125 minus 075 axis 8"` $\rightarrow$ `OD -1.25 -0.75 x 008`, and ASR headers `"Losses measures right -1?"` $\rightarrow$ `Glasses measured: OD -1.00`
     - Multi-turn contralateral refraction stitching: Correctly normalizes split turns (`"-3 - 1."` $\rightarrow$ `OS -3.00 -1.00`, `"120."` $\rightarrow$ `x 120`, `"-325."` $\rightarrow$ `OS -3.25.`, `"3.25."` $\rightarrow$ `OS -3.25.`, `"2.5 -, .75 at 90"` $\rightarrow$ `OD -2.50 -0.75 x 090`) while maintaining turn-by-turn timestamps and dialogue boundaries
   - **Spoken Number Reshaping (`spoken_numbers.py`)**:
     - Signed whole and compound diopters: `"minus two fifty"` $\rightarrow$ `-2.50`, `"minus zero seventy-five"` $\rightarrow$ `-0.75`, `"minus two"` $\rightarrow$ `-2.00` at the end of an eye clause
     - Collapsed ASR quarters: `"minus 250"` $\rightarrow$ `-2.50`, `"minus 75"` / `"minus 075"` / `"minus 0 75"` $\rightarrow$ `-0.75`. A bare `75` stays `75`
     - Whole and compound reading adds: `"plus two add"` $\rightarrow$ `plus 2.00 add`, `"plus two fifty add"` $\rightarrow$ `plus 2.50 add`
     - Guarded fractional diopters: `"minus a quarter diopter"` $\rightarrow$ `-0.25 diopter`, `"plus half sphere"` $\rightarrow$ `+0.50 sphere`
     - Contextual axis numbers and positional digit spelling: `"-0.75 at one eighty"` $\rightarrow$ `-0.75 axis 180`, `"axis one zero five degrees"` $\rightarrow$ `axis 105 degrees`. A short filler may sit between the preposition and the number (`"at the ten degrees"` $\rightarrow$ `axis 10`), and `"after 170 degrees"` (French *après*, Italian *dopo*, Spanish *después*, German *nach*) is an axis only when it follows a cylinder. Appointment times and keratometry meridians stay untouched
     - Reading add with a filler: `"add the +2"` $\rightarrow$ `add +2.00` (the add word stays in front so the refraction grammar still sees it)
     - Snellen acuity word pairs with clinical neighbor tests: `"VA twenty twenty"` $\rightarrow$ `VA 20/20`, `"twenty twenty five"` $\rightarrow$ `20/25`, `"unaided six twelve"` $\rightarrow$ `unaided 6/12`
     - Spoken intraocular pressure integers: `"pressure sixteen both eyes"` $\rightarrow$ `pressure 16 both eyes`
   - **Phrase Indexing and Near-Miss Rewrite Preprocessor (`phrase_repair.py`)**:
     - Prototype surface extraction: Reduces immutable regex patterns from static phrase lexicons and dynamic caller-scoped training aliases into prototype surfaces.
     - Sliding-window fuzzy similarity matching ($\ge 0.93$ threshold) with initial-letter pruning and token-overlap filtering.
     - ASR compound split resolution: `"extra ocular movements in tact"` $\rightarrow$ `EOMI`.
     - Clinical negation protection: strictly prevents rewriting negated expressions (e.g. `"extraocular movements not intact"`, `"denies pain in both eyes"`) into normal findings.
   - **Non-Destructive Bidirectional Coordinate Tracking (`SpanCoordinateMapper`)**: Dynamically records all character offset shifts during preprocessing, seamlessly mapping extracted candidate spans back to exact raw transcript character coordinates to ensure 0.00% Word Error Rate (WER) verbatim narrative preservation outside replaced clinical slots.

    - **Master Clinical Exam Catalog & Intelligent ASR Matcher (`exam_catalog.py`)**:
      - Master taxonomy of 26 core optometric & ophthalmic examinations across 7 clinical categories (Refraction & Vision, Tonometry & Glaucoma, Anterior Segment, Binocular & Motility, Pupils, Posterior Segment, Contact Lens) indexed with LOINC, SNOMED-CT, CPT, canonical nomenclature, and acoustic speech variations. Catalog LOINC codes match the FHIR observation constants, including tear breakup time `71811-4`, pupils `80315-5`, and cup-to-disc `70949-3`.
      - **Intelligent Phonetic & Fuzzy Sequence Matcher (`intelligent_match_exam`, `lookup_exam(..., fuzzy=True)`)**:
        - Resolves unscripted and unseen speech recognition phonetic slips to their canonical clinical procedures via SequenceMatcher fuzzy ratio with calibrated threshold gating ($\ge 0.78$):
          - *"manifest refractionist"* / *"manif refractor"* / *"manif refraction"* / *"manifest fractor"* $\rightarrow$ `manifest refraction` (SNOMED `252886007` / CPT `92015`)
          - *"manifestion refraction"* / *"money refraction"* / *"many refraction"* / *"manifast refraction"* $\rightarrow$ `manifest refraction`
          - *"visual quality"* / *"visual equity"* $\rightarrow$ `visual acuity` (LOINC `8629-0`, distance visual acuity)
          - *"tomate meter"* / *"tomato meter"* / *"tonomiter"* $\rightarrow$ `tonometer` (LOINC `55284-4` / SNOMED `252859005`)
          - *"carrot ometry"* / *"care atometry"* $\rightarrow$ `keratometry` (LOINC `8626-6` / SNOMED `252882009`)
          - *"pack symmetry"* / *"pachy metry"* / *"pocket metry"* $\rightarrow$ `pachymetry` (LOINC `71813-0` / CPT `76514`)
          - *"split lamp"* / *"slit lab"* $\rightarrow$ `slit lamp` (SNOMED `252800007` / CPT `92004`)
          - *"fundus copy"* $\rightarrow$ `fundoscopy` (SNOMED `252832004` / CPT `92201`)
          - *"psycho plegic"* / *"cycle plegic"* $\rightarrow$ `cycloplegic` (SNOMED `252888008`)
          - *"nuclear balance"* / *"binocular ballast"* $\rightarrow$ `binocular balance` (SNOMED `252891004`)
          - *"gonio scopy"* $\rightarrow$ `gonioscopy` (SNOMED `76949005` / CPT `92020`)
          - *"written copy"* / *"retina scopy"* $\rightarrow$ `retinoscopy` (SNOMED `39155005`)
        - **Strict Negative Discrimination**: Non-exam phrases (e.g. *"the patient spent money on glasses"*, *"blood pressure check"*, *"patient has manifest anxiety"*) yield scores below threshold and are never falsely modified (0.00% false replacement rate).
      - The preprocessor calls the matcher only after the pinned acoustic regexes, so gold strings such as `tonometer` and `manifest refraction` stay put. Running text uses a stricter compound cutoff ($\ge 0.92$) than isolated lookup ($\ge 0.78$). Already-correct spacing variants (`auto refraction`, `auto refractor`) are aliases and are not rewritten. Residual slips such as *"gonyo scopy"* still become `gonioscopy`.
      - Seamlessly anchors downstream measurement parsing (e.g., *"I focus now on the manifestion refraction minus 2.00 minus 0.50 axis 180 for the right eye"* $\rightarrow$ `I focus now on the manifest refraction OD -2.00 -0.50 x 180`) while repairing standalone exam announcements into standardized terminology without modifying adjacent narrative text.
    - **Anatomy Collocation Matcher (`anatomy_catalog.py`)**: Rewrites a structure or finding only as a collocation, never a diagnosis standing alone. *"corners otherwise clear"* $\rightarrow$ `corneas otherwise clear`, *"conjunctivitis quiet"* $\rightarrow$ `conjunctiva quiet`, *"interior chamber"* $\rightarrow$ `anterior chamber`, *"my bohemian glands"* $\rightarrow$ `meibomian glands`, *"nuclear salad"* $\rightarrow$ `nuclear sclerosis`, *"drew zen"* / *"drew sin"* $\rightarrow$ `drusen`. *"viral conjunctivitis"*, *"corners of the room"*, and *"nuclear family"* stay verbatim. Grading regexes then see the canonical English.
    - **Ocular Medication Matcher (`medications_catalog.py`)**: Small pinned catalog (latanoprost, timolol, brimonidine, dorzolamide, prednisolone / Pred Forte, proparacaine, tropicamide, Systane, aflibercept, ranibizumab) with drug class and RxNorm ingredient CUIs. Short tokens match only inside an ocular window (`drop`, `eye`, `OU`/`OD`/`OS`, `bid`/`qd`). *"latte no prost drops"* $\rightarrow$ `latanoprost`. *"six stain on the shirt"* and a person named Tim stay verbatim. French, German, Italian, and Spanish aliases cover structure names and INN drug names; English number homophones do not run on those locales.
    - **English Number & Laterality Homophones (`number_words.py`, `refraction_repair.py`)**: Context-gated, English only. `minus`/`plus` + `to`/`too` + a diopter tail becomes `two`; `for` in the same frame becomes `four`; rapid refraction triplets where a signed sphere (or plano) is followed by `to`/`too`/`for` and an integer axis (`minus 1.5 to 15` $\rightarrow$ `minus 1.5 2 15` $\rightarrow$ `OS -1.50 -2.00 x 015`) become `2` or `4`; `axis` + `won` + a tens word becomes `one`; `right`/`left` + `guy` becomes `eye` only beside a refraction or acuity token; `oh dee` / standalone `ode` becomes `OD` as laterality. *"I went to fifty appointments"*, *"I went to 15 appointments"*, *"visual acuity 20 to 20"*, *"plus for the patient"*, *"axis won the game"*, and *"he owed"* stay verbatim. Every substitution is recorded on `SpanCoordinateMapper` so `original_text[raw_start:raw_end]` equals the spoken span.


5. **Reversible Audit Trail**:
   - Performs non-destructive in-place character offset span replacement.
   - Produces a granular audit log recording `start`, `end`, `original_text`, `replacement_text`, `finding_type`, `rule_or_model_id`, and clinician review status.

6. **HL7 FHIR R4 Discrete Observation Synchronization**:
   - Automatically generates structured FHIR R4 Observation resources using standard clinical vocabularies:
     - **Distance Visual Acuity**: LOINC `8629-0`
     - **Near Visual Acuity**: LOINC `8630-8`
     - **Correction Method**: SNOMED-CT `422490008` (_Without corrective lenses / sc_) / `420130008` (_With corrective lenses / cc_)
     - **Refraction (Sphere, Cylinder, Axis, Add, PD)**:
       - **Composite Refraction Observation (US Core & Epic profile)**: LOINC `28618-7` (*Physical findings of Refraction*) with subcomponents for Sphere (`28615-3`), Cylinder (`28616-1`), Axis (`28617-9`), Reading Add (`28614-6`), and Pupillary Distance (`89073-1`), paired with SNOMED-CT laterality (`362502000` OD, `362503005` OS, `40638003` OU).
       - **Legacy Refraction Panel**: LOINC `28634-4` with UCUM `[diop]` / `deg`.
     - **Contact Lens Prescription & Evaluation**: LOINC `57077-0` (*Contact lens prescription*), Base Curve (`57078-8`), Diameter (`57079-6`), and Over-Refraction (`95383-6`).
     - **Prism Prescription**: LOINC `28641-9` with UCUM `[p'diop]`
     - **Keratometry Curvature Panel**: LOINC `8626-6` (Flat K LOINC `8627-4`, Steep K LOINC `8628-2`). Principal meridians are degree components on the same Observation (`flat-meridian`, `steep-meridian`, UCUM `deg`) from `urn:optonorm:fhir:CodeSystem:keratometry-meridian`. Panel `8626-6` has no axis child. LOINC panel `95298-6` lists laterality-specific Axis and Axis 2 (`28975-1`, `28977-7`, `28985-0`, `28987-6`), which are not flat versus steep meridians, so those codes are not used.
     - **Corneal Pachymetry (CCT)**: LOINC `71813-0` with UCUM `um`
     - **Tear Breakup Time (TBUT)**: LOINC `71811-4` with UCUM `s`
     - **Intraocular Pressure (Tonometry)**: LOINC `55284-4` with UCUM `mm[Hg]`
     - **Cup-to-Disc (C/D) Ratio**: LOINC `70949-3`
     - **Pupillary Reflex & Diameters**: LOINC `80315-5` (Pupillary reflex), LOINC `80313-0` (Pupil diameter in `mm`), LOINC `76504-0` (RAPD)
     - **Biomicroscopy & Cataract Grading**: SNOMED-CT `414646002` (Nuclear sclerosis), LOINC `70950-1` (AC flare), SNOMED-CT `231872005` (SPK)
     - **Strabismus & Binocular Alignment**: LOINC `70951-9` with UCUM `[p'diop]`
     - **Laterality**: SNOMED-CT `28400003` (Right eye), `28400004` (Left eye), `28400005` (Both eyes)
     - **Diagnostic Document Packaging**: LOINC `11528-7` (DiagnosticReport) packaged into FHIR Composition and Consultation Document Bundles
     - **Ocular Medications**: FHIR R4 `MedicationStatement` with an RxNorm coding (`http://www.nlm.nih.gov/research/umls/rxnorm`) when the catalog entry has an ingredient CUI. Systane is exported as text only. Other findings stay Observations.
   - *Note: `http://loinc.org`, `http://snomed.info/sct`, and `http://unitsofmeasure.org` are canonical namespace URIs, not network endpoints. OptoNorm operates 100% offline with zero external network calls or API token requirements.*

7. **Multilingual (i18n) Foundation & French / Italian / Spanish / German Clinical Providers**:
   - **Pluggable `LocaleProvider` Interface**: Isolates locale-specific dictionaries (spoken number words, ASR elisions, laterality synonyms, and clinical keywords) from core parsing logic.
   - **French Clinical Language Provider (`FrenchLocaleProvider`)**: Complete support for European and Canadian French clinical transcripts. Supports French vigesimal spoken numbers (`soixante-dix`, `quatre-vingts`, `quatre-vingt-dix`, Swiss/Belgian `septante`, `nonante`), decimal dictations (`deux virgule cinquante`, `moins deux cinquante`, `et demi`), Monoyer distance visual acuity (`10/10` to `1/20`) and Parinaud near visual acuity (`P1.5` to `P14`), French tonometry, keratometry, corneal pachymetry, TBUT, prism base directions (`BT` $\rightarrow$ `BO`, `BN` $\rightarrow$ `BI`, `BS` $\rightarrow$ `BU`; bare `bi` is `BD` in vertical context, `BI` in horizontal context, and reviewed when ambiguous), and French clinical abbreviations.
   - **Italian Clinical Language Provider (`ItalianLocaleProvider`)**: Complete support for Italian clinical transcripts. Supports Italian spoken compound numbers (`quarantatré`, `cinquecentoquaranta`), decimals (`virgola`, `due cinquanta`, `zero settantacinque`, `e mezzo` / `e mezza`), Decimi distance visual acuity (`10/10` to `1/20`, modifiers `-2`, `+1`, pinhole `al foro stenopeico`), Jaeger near reading scale (`J1` to `J7`), Italian subjective refraction (`sfera`, `cilindro`, `asse`, `addizione per vicino`), Italian tonometry (`pressione intraoculare`, `tono oculare`, `PIO`), ultrasound corneal pachymetry (`pachimetria corneale`, `spessore corneale centrale`), tear breakup time (`tempo di rottura del film lacrimale`, `break up time`), keratometry (`cheratometria`, `curvatura corneale`), prism bases (`base esterna` / `base temporale` $\rightarrow$ `BO`, `base interna` / `base nasale` $\rightarrow$ `BI`, `base superiore` $\rightarrow$ `BU`, `base inferiore` $\rightarrow$ `BD`; bare `bi` is `BD` in vertical context, `BI` in horizontal context, and reviewed when ambiguous), and Italian clinical abbreviations.
   - **Spanish Clinical Language Provider (`SpanishLocaleProvider`)**: Complete support for Spanish clinical transcripts. Supports Spanish cardinal and compound numbers (`cuarenta y tres`, `quinientos cuarenta`, `ciento ochenta`), decimals (`coma`, `con`, `dos cincuenta`, `cero setenta y cinco`, `y medio` / `y media`), Décimas distance visual acuity (`10/10` to `1/20`, modifiers `-2`, `+1`, pinhole `al agujero estenopeico`), Jaeger near reading scale (`J1` to `J7`), Spanish subjective refraction (`esfera`, `cilindro`, `eje`, `adición para cerca`, `plano`), Spanish tonometry (`presión intraocular`, `tono ocular`, `PIO`), ultrasound corneal pachymetry (`paquimetría corneal`, `espesor corneal central`), tear breakup time (`tiempo de rotura de la película lagrimal`, `tbut`), keratometry (`queratometría`, `curvatura corneal`), prism bases (`base temporal` / `base externa` $\rightarrow$ `BO`, `base nasal` / `base interna` $\rightarrow$ `BI`, `base superior` $\rightarrow$ `BU`, `base inferior` $\rightarrow$ `BD`; bare `bi` is `BD` in vertical context, `BI` in horizontal context, and reviewed when ambiguous), and Spanish clinical abbreviations.
   - **German Clinical Language Provider (`GermanLocaleProvider`)**: Complete support for German clinical transcripts. Supports German inverted compound numbers (`einundzwanzig`, `fünfundvierzig`, `einhundertachtzig`, `fünfhundertvierzig`), decimals (`Komma`, `zwei fünfzig`, `null fünfundsiebzig`, `einundeinhalb` / `eineinhalb`, `einviertel`, `dreiviertel`), DIN 58220 decimal Visus (`1,0` to `0,05`, modifiers `-2`, `+1`, pinhole `mit stenopäischer Lücke`), Nieden (`N1` to `N8`), Birkhäuser (`B1` to `B6`), and Jaeger (`J1` to `J7`) near reading scales, German subjective refraction (`Sphäre`/`Sph`, `Zylinder`/`Zyl`, `Achse`/`A`, `Nahzusatz`/`Add`), German tonometry (`Augeninnendruck`, `IOD`, `Applanationstonometrie`), ultrasound corneal pachymetry (`Hornhautdicke`, `Pachymetrie`, `CCT`), tear breakup time (`Tränenfilm-Aufreißzeit`, `TBUT`), keratometry (`Hornhautkrümmung`, `Ophthalmometrie`), prism base directions (`Basis temporal`/`außen` $\rightarrow$ `BO`, `Basis nasal`/`innen` $\rightarrow$ `BI`, `Basis oben` $\rightarrow$ `BU`, `Basis unten` $\rightarrow$ `BD`), German laterality (`RA`/`LA`/`BA`), and German clinical abbreviations.
   - **Abstract Parametric Grammar Templates (`templates.py`)**: Defines universal medical syntax topologies once across Western languages, binding lexical token sets dynamically with compiled regex caching for zero latency overhead (0.38 ms).
   - **Zero-Dependency Clinical Language Detection (`detector.py`)**: Automatically infers transcript language (`en`, `fr`, `it`, `es`, `de`) via high-speed clinical keyword heuristics when invoked with `locale=Locale.AUTO`.
   - **Configurable Output Conventions**: Supports both `ShorthandConvention.INTERNATIONAL` (standard Anglo-Latin shorthand `OD`, `OS`, `OU`, `IOP: ... mmHg`) and `ShorthandConvention.LOCALIZED`.
   - **Seamless Extensibility**: Adding a new language requires zero code changes to grammars, guards, or pipeline—only authoring a single `LocaleProvider` plugin.

8. **Production FastAPI REST API & Plug-and-Play APIRouter**:
   - **Dual-Mode Consumption**: Runs as an independent containerized microservice or embeds directly into external FastAPI applications via `app.include_router(get_optonorm_router(...))`.
   - **Standard REST Endpoints**: `/v1/normalize` (with granular audit edits and latency telemetry), `/v1/fhir` (HL7 FHIR R4 Bundle generation), `/v1/locales` (supported scales and conventions), `/v1/contact/token` (anti-bot HMAC-SHA256 time tickets), `/v1/contact` (public inquiry channel with honeypot & PHI guards), and `/v1/health`.
   - **Content Negotiation & Telemetry**: Automatically infers target language from `Accept-Language` headers when not explicitly provided and reports sub-millisecond execution latency in `X-Process-Time-Ms` response headers.
   - **Interactive Documentation**: Auto-generated Swagger UI (`/docs`) and ReDoc (`/redoc`) with pre-configured clinical examples across all supported languages.

9. **Real-Time Streaming & Ambient Scribing (Chunked ASR)**:
   - **Sliding Window Buffer (`StreamingNormalizerBuffer`)**: Handles live ASR chunk feeds and commits a finding only at a real sentence boundary (`.?!` or a newline) or on flush. Conjunctions (`and`, `et`, `e`, `y`, `und`) and fixed phrases stay pending, so a bilateral measurement or an optional phrase tail is normalized with the rest of the sentence. A pause holds an unfinished diopter (`"minus two"`), a partial axis after a power (`"at one"`), and an IOP cue that is still waiting for a number. Finished speech is committed, including `"blood pressure"`, `"looking at"`, and `"see you at ten"`.
   - **Symmetrical Bilateral Holdback**: Automatically holds back uncommitted unilateral findings (IOP, Pachymetry, TBUT) across speech pauses, waiting for potential contralateral clauses before committing canonical binocular `OU` shorthand or monocular `OD`/`OS`.
   - **Bidirectional Offset Tracking (`OffsetTracker`)**: Maintains coordinate mappings between raw incoming ASR stream offsets and normalized shorthand indices across non-destructive span edits, allowing downstream UI components to map audio-aligned timestamps to transformed text.
   - **Zero-Latency Narrative Flushing**: Immediately emits conversational ambient prose outside clinical slots without buffering delays, preserving doctor-patient dialogue 100% verbatim.
   - **Flush Reconcile**: Ending the session runs `normalize()` once on the full raw transcript and emits a `reconcile` event with that note, the batch edits, the detected language, clinical alerts, and a FHIR R4 transaction bundle (`include_alerts`, composite refraction). The finished stream matches the batch form. The playground waits up to 30 seconds for that event. If the socket closes first, the page says the connection closed.
   - **Native Streaming Transport**: Full support for real-time WebSockets (`/v1/normalize/stream`), Server-Sent Events (`/v1/normalize/sse`), and piped CLI streaming (`optonorm --stream`).

10. **Interactive Clinical Web Playground (React 19 & Redux Toolkit)**:
    - **Modern Scribe Cockpit**: High-fidelity clinical workbench powered by React 19, Redux Toolkit, TanStack Form, and TanStack Query with zero state-drift and strict single-direction data flow.
    - **Simulated Ambient ASR Scribe**: Full-duplex WebSocket stream (`/v1/normalize/stream`) simulating live speech token arrival, featuring a visual **Holdback HUD** displaying bilateral symmetry pauses (e.g. IOP / Pachymetry / TBUT) before canonical `OU` commit.
    - **Visual Clinical Diff Viewer & Findings Table**: Side-by-side and inline comparative diff highlighting replaced speech spans vs untouched verbatim narrative, coupled with structured extraction tables mapped to LOINC and SNOMED CT terminology.
    - **HL7 FHIR R4 Inspector & US Core 6.1.0 Consultation Note**: Interactive JSON tree viewer for discrete Observation bundles alongside rich human-readable XHTML consultation notes with one-click export (JSON / HTML) and print-ready layouts.

11. **Clinical Decision Support (CDS) & Sight-Threatening Red-Flag Triage Engine**:
    - **Glaucoma Asymmetry Alert**: Detects cup-to-disc ratio asymmetry $\ge 0.20$ (`CDS_GLAUCOMA_ASYMMETRY`, Warning; Critical Urgent when asymmetry $\ge 0.30$ or either ratio $\ge 0.85$). The rationale says “vertical” only when both vertical meridians were parsed, and “scalar” for a pair of scalar ratios.
    - **Enlarged Cup Alert**: Flags a scalar ratio, or a parsed vertical meridian, at the shared $\ge 0.65$ cutoff (`CDS_GLAUCOMA_LARGE_CUP`, Warning). The rationale names “scalar” or “vertical” to match the component that was parsed.
    - **Hyper-Acute IOP / Angle-Closure Alert**: Flags intraocular pressure $\ge 30\text{ mmHg}$ (`CDS_ELEVATED_IOP_CRITICAL`, Critical Urgent).
    - **Retinal Detachment / Tear Suspect**: Flags flashes paired with floaters, or a curtain/veil (`CDS_RETINA_TEAR_SUSPECT`, Critical Urgent).
    - **Acute Floater Alert**: Flags floaters qualified by new, sudden, or shower (`CDS_NEW_FLOATERS_WARNING`, Warning). Chronic floaters alone do not alert.
    - **Pediatric Amblyopia Risk**: Flags spherical equivalent anisometropia $> 1.50\text{ D}$ in pediatric patients (`CDS_AMBLYOPIA_RISK`, Warning).
    - **Beta-Blocker and Asthma / COPD**: A timolol (beta-blocker) medication slot plus asthma or COPD in the original transcript raises `CDS_BETA_BLOCKER_ASTHMA` (Warning). The alert does not rewrite the narrative.
    - Synchronizes alerts as FHIR `Flag` resources conforming to US Core standards.

12. **Epic Hyperspace & EHR Dotphrase Macro Exporter (`.OPTOEXAM`)**:
    - Generates multi-line EHR smartphrases ready for direct copy/paste or API insertion into Epic Hyperspace, Cerner Millennium, or AthenaHealth.
    - CLI flag `--dotphrase` and API parameter `export_dotphrase=true`.

---

## Architectural Overview

```mermaid
flowchart TD
    subgraph Client ["Interactive Clinical Web Playground (React 19 + Redux Toolkit)"]
        W1[Clinical Form & Presets<br/>@tanstack/react-form]
        W2[Ambient Scribe Simulator<br/>WebSocket Stream & Holdback HUD]
        W3[Clinical Diff & Findings<br/>LOINC / SNOMED CT Table]
        W4[FHIR R4 Inspector & Note<br/>US Core 6.1.0 Consultation Note]
        W5[Optical Tools Suite<br/>Cyl Transposition · Vertex · Spherical Equivalent]
        W6[Community & Contributing Portal<br/>Persona Inquiries · Secure Zero-PHI Contact Hub]
        W7[Training Workbench<br/>/training · phrase mapped to a known shorthand]
    end

    subgraph Service ["FastAPI Microservice Engine (Port 8000)"]
        API[FastAPI Gateway<br/>/v1/normalize · /v1/fhir · /v1/normalize/stream · /v1/contact · /v1/training]
        AUTH[Authentication and audit<br/>JWT sessions, API keys, role scopes]
        CONTACT[Secure Contact Hub & Anti-Bot<br/>HMAC Time Gate · Nonce Replay Guard · Sliding Rate Limit · 90d Prune]
        B[Clinical Locale Provider & Language Inference<br/>en, fr, it, es, de]
        PRE[Acoustic & Speech Dysfluency Preprocessor Pipeline<br/>Self-Correction · Punctuation Re-Stitcher · Stutter Collapser<br/>Pinned Acoustic Repair 2.0 · Locale Exam, Anatomy & Medication Matchers<br/>English & French Homophones · Refraction Stages & Diopter Repair]
        C[Candidate Span Extractors]
        
        subgraph Grammars & Lexicon
            C --> D1[Visual Acuity Grammar<br/>Snellen, Metric 6m, Pinhole, Decimals]
            C --> D2[Refraction Grammar<br/>Sphere, Cyl, Axis, Add, RefractionStage]
            C --> D3[Tonometry / IOP Grammar<br/>GAT, NCT, iCare, Spoken Units]
            C --> D4[Prism Grammar<br/>Δ BI, BO, BU, BD]
            C --> D5[Keratometry Grammar<br/>K Flat, Steep, Cyl, Axis]
            C --> D6[Cornea Grammar<br/>CCT in µm, TBUT in s]
            C --> D7[Dynamic C/D Ratio Grammar<br/>Scalar, Asymmetric, Biaxial, Notching]
            C --> D8[Pupil & RAPD Grammar<br/>Diameters, ND RAPD, LND, Shape]
            C --> D9[Slit Lamp Grading Grammar<br/>LOCS III Cataract, SUN AC Cells/Flare, SPK]
            C --> D10[Strabismus & Alignment Grammar<br/>Cover Test, Hirschberg, NPC, Duochrome]
            C --> D11[Fixed Clinical Phrase Lexicon]
            C --> D12[Contact Lens Grammar<br/>Fit, Rotation, Vault, Over-Refraction]
            C --> D13[Master Catalogs & Intelligent Matchers<br/>Exams, Anatomy, Medications & Contact Lenses]
            C --> D14[Retina & Posterior Segment Grammar<br/>Optic Disc Sharpness, Drusen/AMD, Retinopathy]
            C --> D15[Dry Eye / DEWS II Grammar<br/>Schirmer Test, MGD Expressibility Grade, TMH]
            C --> D16[Gonioscopy Grammar<br/>Shaffer, Spaeth, Scheie, PAS]
            C --> D17[Perimetry Grammar<br/>HVF 24-2/30-2/10-2, MD, PSD, VFI, GHT]
            C --> D18[OCT Grammar<br/>CST, RNFL, IRF/SRF, Normative Status]
            C --> D19[Motility Grammar<br/>EOMI, FROM, Gaze Grades, CN Palsy]
            C --> D20[Stereopsis Grammar<br/>Arcseconds, Titmus, Worth 4-Dot]
            C --> D21[Confrontation Grammar<br/>FTFC, Quadrants, Hemianopia]
            C --> D22[Topography Grammar<br/>Sim K, Kmax, I-S, BAD-D]
        end

        D1 & D2 & D3 & D4 & D5 & D6 & D7 & D8 & D9 & D10 & D11 & D12 & D13 & D14 & D15 & D16 & D17 & D18 & D19 & D20 & D21 & D22 --> E[Post-Hoc Verification Pipeline]

        subgraph Safety Guards
            E --> F1[Numeric Entailment Guard<br/>Zero Invented Numbers, Multilingual & Roman Numerals]
            E --> F2[Physiological Range Guard<br/>0.25D, 1-180° Axis, CCT, TBUT, C/D, Pupils, Grading, Δ, Spaeth, HVF, OCT]
        end

        F1 & F2 --> G[Conflict & Overlap Resolver<br/>Priority-Ranked Non-Overlapping Spans]
        TRAIN["Trained phrase rules<br/>Built-in (tier 0) → Personal (tier 1) → Shared (tier 2)"]
        TRAIN --> G
        G --> H[Residue Detection & Optional SLM Escalation]
        H --> I[Reversible In-Place Span Replacement Engine]
    end

    Client <-->|"REST, WebSocket, Bearer JWT or X-API-Key<br/>Host :3000 → Container :8080 / :8000"| API
    API --> AUTH --> B --> PRE --> C
    API --> TRAIN
    API --> CONTACT
    PRE -.->|"Exact Raw Span Offsets (SpanCoordinateMapper)"| I

    I --> J[Normalized EHR Clinical Text]
    I --> K[Structured FHIR R4 Bundle Exporter<br/>VisionPrescription · Observations · ICD-10-CM · SNOMED CT]
    H --> L[Clinician Correction Harvesting JSONL]
```

A trainer or clinician teaches a phrase the pipeline missed, such as `gonioscopy Shaffer grade of 4 OU`, and pairs it with a shorthand the normalizer already emits, `Shaffer 4 OU`. The same rule also accepts a short hesitation between the words, including `gonioscopy Shaffer grade of, ehmm, 4 OU`. Words the trainer typed stay required. A grade and an eye become slots. The rule does not invent a new canonical, and it does not change a gold-set item.

### Training Scopes & Precedence

OptoNorm supports two distinct levels of training vocabulary:
- **Shared Training Rules (`vocabulary:train`)**: Managed by trainers, administrators, and superusers. These rules apply globally across all normalization requests (`owner_user_id IS NULL`) and undergo gold-set conflict verification.
- **Personal Training Phrases (`vocabulary:personal`)**: Accessible to clinicians, scribes, and trainers. Allows clinicians to teach custom, doctor-specific phrases that apply strictly to their own transcripts (`owner_user_id = principal.id`). Personal rules skip the baseline gold-set gate and never contaminate other clinicians' transcripts or process-wide catalogs (`MEDICATION_CATALOG`, etc.).

During transcript normalization, rules are dynamically composed and resolved using a 3-tier precedence hierarchy:
1. **Tier 0 (Built-in Grammars)**: Native typed parsers always win over learned rules when spans conflict.
2. **Tier 1 (Personal Training Rules)**: The authenticated clinician's personal phrases (`caller_context(principal.id)`) take precedence over global shared rules.
3. **Tier 2 (Shared Training Rules)**: Baseline community rules available to all callers.

When no user is authenticated or an unauthenticated CLI/API client calls the normalizer, only shared rules apply.

A taught phrase can mark individual words as optional and can collect several phrases that resolve to the same shorthand. The teach form sends the locale used to preprocess the example. Catalog aliases are registered again when the process starts. Grammar skeletons stay in code. Extra clinical literals (an IOP anchor, a laterality phrase, a prism base) are stored in `grammar_tokens`, escaped, and spliced into the existing template after a named-group, latency, and gold-set check. `POST /v1/training/vocabulary` publishes one. Deleting the row restores the shipped vocabulary.

Clinicians and scribes can also highlight un-normalized text directly within the **Clinical Spoken / Dictated Transcript** or **Live Stream** (when paused/stopped and not recording mic) to trigger a contextual floating action pill (`✨ Teach Phrase: "..."`) or a toolbar button that transfers the highlighted text into the rule inducer with automatic encounter locale synchronization.

Both the `/training` rule workbench and the User Administration panel feature headless, sortable, paginated data grids powered by **TanStack Table (`@tanstack/react-table`)** with Redux-controlled state (zero `useState`), global multi-field search, finding-type category faceting, and batch selection.

---

## HL7 FHIR R4 Architecture & Terminology Systems (LOINC, SNOMED CT, UCUM)

OptoNorm bridges the gap between conversational clinical transcripts and standardized healthcare informatics by generating valid **HL7 FHIR R4** resources.

### 1. Canonical Namespace URIs vs. Network Fetching

> [!IMPORTANT]
> **Zero Network Overhead & Offline Operation**:  
> OptoNorm **does NOT fetch data from `http://loinc.org`, `http://snomed.info/sct`, or `http://unitsofmeasure.org` over the internet at runtime**. No API tokens, keys, or external network connections are required to normalize transcripts or export FHIR JSON bundles.

In HL7 FHIR R4, medical concepts are identified using `Coding` elements comprising a `system`, a `code`, and an optional `display`:

$$\text{Coding} = \langle \text{System URI}, \text{Code}, \text{Display} \rangle$$

The URIs are **globally unique canonical namespace identifiers**, not HTTP endpoints to query:

```json
{
  "system": "http://loinc.org",
  "code": "8629-0",
  "display": "Visual acuity distance"
}
```

The URI `http://loinc.org` explicitly declares to receiving EHR systems (e.g., Epic, Cerner, Apple Health, HAPI FHIR) that the code `8629-0` originates from the international LOINC dictionary, avoiding semantic ambiguity with internal hospital charge codes.

### 2. Terminology Governance & Licensing

| System Name | Canonical System URI | Governing Organization | Purpose in OptoNorm | Licensing & Access Model |
|---|---|---|---|---|
| **LOINC** | `http://loinc.org` | Regenstrief Institute | Clinical measurements, panels, exam observations (Visual Acuity, Refraction, Tonometry, CCT) | **Free worldwide license** under Regenstrief terms. Free to embed in software products without fees or tokens. |
| **SNOMED CT** | `http://snomed.info/sct` | SNOMED International | Anatomical sites & laterality (`Right eye`, `Left eye`), qualitative findings, correction state | **Free in Member Countries** (US, UK, Germany, Canada, Australia, Spain, Netherlands, Switzerland, etc.). Foundational concept references are royalty-free. |
| **UCUM** | `http://unitsofmeasure.org` | Regenstrief / UCUM Organization | Standardized clinical units of measure (`[diop]`, `deg`, `mm[Hg]`, `um`, `s`, `[p'diop]`) | **Public domain / Open Source**. Completely free to use without registration or fees. |
| **Keratometry meridians** | `urn:optonorm:fhir:CodeSystem:keratometry-meridian` | OptoNorm | Flat and steep principal meridians (`flat-meridian`, `steep-meridian`) in degrees on the keratometry Observation | Local code system, embedded offline. Used because LOINC panel `8626-6` has no axis member. |
| **Glaucoma diagnostics** | `urn:optonorm:fhir:CodeSystem:glaucoma-diagnostics` | OptoNorm | Perimetry indices (`mean-deviation`, `pattern-standard-deviation`, `visual-field-index`, `glaucoma-hemifield-test`) and OCT measurements on the gonioscopy, perimetry, and OCT observations | Local code system, embedded offline. Gonioscopy itself is SNOMED `76949005` and perimetry is SNOMED `103752008`. Retinal OCT is CPT `92134`; optic-nerve OCT is CPT `92133`. |
| **Binocular vision** | `urn:optonorm:fhir:CodeSystem:binocular-vision` | OptoNorm | Motility grades, stereoacuity in arcseconds, Worth 4-dot result, and confrontation quadrants or defects | Local code system, embedded offline. Ocular motility is LOINC `79803-3`. Stereotests are SNOMED `252853008` and Worth four-dot test is SNOMED `415850001`. Confrontation visual field test is SNOMED `410560002`. |
| **Contact lens fit** | `urn:optonorm:fhir:CodeSystem:contact-lens-fit` | OptoNorm | Multifocal add, toric rotation, push-up, movement, vault, fluorescein pattern, and spherical or spherocylindrical over-refraction | Local code system, embedded offline. Used for dynamic fit findings. |
| **Corneal topography** | `urn:optonorm:fhir:CodeSystem:corneal-topography` | OptoNorm | Sim K, Kmax, I-S, BAD-D, and cone morphology on the topography Observation | Local code system, embedded offline. The procedure is SNOMED `252830007` and CPT `92025`. |

### 3. Internal Architecture & Zero-Token Runtime

OptoNorm operates as a **deterministic producer/generator** of FHIR resources:

```mermaid
flowchart LR
    A[Raw Clinical Transcript] --> B[OptoNorm NLP Engine]
    B --> C[Strongly Typed Findings<br/>Pydantic Data Models]
    C --> D[fhir.py / fhir_document.py]
    
    subgraph Statically Compiled Constants
        D --> E1[LOINC_SYSTEM = 'http://loinc.org']
        D --> E2[SNOMED_SYSTEM = 'http://snomed.info/sct']
        D --> E3[UCUM_SYSTEM = 'http://unitsofmeasure.org']
    end
    
    D --> F[HL7 FHIR R4 Bundle<br/>Transaction or Document]
    F --> G[EHR Ingestion API<br/>Epic / Cerner / HAPI FHIR]
```

1. **Extraction**: Clinical entities are parsed into strongly-typed slot models (`VisualAcuitySlots`, `RefractionSlots`, `IOPSlots`, `PrismSlots`, `KeratometrySlots`, `PachymetrySlots`, `TBUTSlots`, `CDRatioSlots`, `PupilSlots`, `GradingSlots`, `AlignmentSlots`).
2. **Deterministic Mapping**: In [`fhir.py`](src/opto_normalizer/fhir.py) and [`fhir_document.py`](src/opto_normalizer/fhir_document.py), pre-mapped lookup dictionaries translate findings into standard FHIR R4 `Observation`, `DiagnosticReport`, and `Composition` resources.
3. **Serialization**: In-memory Python dictionaries are serialized directly to JSON strings in **< 0.05 ms** with zero external dependencies.

### 4. When ARE Permissions or Tokens Needed?

Tokens and credentials are only relevant when integrating OptoNorm into an external institutional pipeline:

- **Pushing Bundles to an EHR (SMART on FHIR)**: When sending generated FHIR bundles to a live EHR endpoint (e.g. `POST /Bundle` to Epic or Cerner), the calling application must pass an OAuth2 Bearer Token (`Authorization: Bearer <access_token>`) obtained via SMART-on-FHIR client credentials.
- **Dynamic Terminology Validation ($lookup / $expand)**: If an external server validates codes dynamically against a live Terminology Server (e.g. NLM UMLS Terminology Services or `fhir.loinc.org`), a free UMLS API Key or LOINC account is required by that external server.

---

## Performance & Benchmark Metrics

Evaluated on the 907-utterance Multilingual Gold Evaluation Benchmark across English ([`data/gold_set.json`](data/gold_set.json)), French ([`data/gold_set_fr.json`](data/gold_set_fr.json)), Italian ([`data/gold_set_it.json`](data/gold_set_it.json)), Spanish ([`data/gold_set_es.json`](data/gold_set_es.json)), and German ([`data/gold_set_de.json`](data/gold_set_de.json)):

| Category                           | Gold Items |  Accuracy  | Hallucination Rate | False Replacements | Mean Latency |
| ---------------------------------- | :--------: | :--------: | :----------------: | :----------------: | :----------: |
| **Visual Acuity (US, Metric 6m, Decimals)** | 178 | 100.0% | 0.00% | 0 / 52 (0.00%) | — |
| **Refraction (Sphere, Cyl, Axis, Add)**     | 151 | 100.0% | 0.00% | 0 / 52 (0.00%) | — |
| **Intraocular Pressure (IOP)**     |     97     |   100.0%   |       0.00%        |   0 / 52 (0.00%)   |      —       |
| **Fixed Phrases & Slit Lamp**      |    236     |   100.0%   |       0.00%        |   0 / 52 (0.00%)   |      —       |
| **Prism & Strabismus**             |     55     |   100.0%   |       0.00%        |   0 / 52 (0.00%)   |      —       |
| **Keratometry (K-Readings)**       |     45     |   100.0%   |       0.00%        |   0 / 52 (0.00%)   |      —       |
| **Corneal Pachymetry (CCT)**       |     39     |   100.0%   |       0.00%        |   0 / 52 (0.00%)   |      —       |
| **Tear Breakup Time (TBUT)**       |     37     |   100.0%   |       0.00%        |   0 / 52 (0.00%)   |      —       |
| **Cup-to-Disc**                    |      3     |   100.0%   |       0.00%        |   0 / 52 (0.00%)   |      —       |
| **Grading**                        |      2     |   100.0%   |       0.00%        |   0 / 52 (0.00%)   |      —       |
| **Contact Lens**                   |      2     |   100.0%   |       0.00%        |   0 / 52 (0.00%)   |      —       |
| **Negative Controls & Non-Clinical** |   52     |   100.0%   |       0.00%        |   0 / 52 (0.00%)   |      —       |
| **Alignment (NPC, AC/A, NRA/PRA, cover test)** |      6     |   100.0%   |       0.00%        |   0 / 52 (0.00%)   |      —       |
| **Phrase repairs**                 |      4     |   100.0%   |       0.00%        |   0 / 52 (0.00%)   |      —       |
| **Overall Multilingual System**    |  **907**   | **100.0%** |     **0.00%**      |     **0.00%**      | **1.63 ms**  |

### Slot-Level Information Extraction & Verbatim Integrity Metrics

Evaluated across all 1,498 individual clinical slot values extracted across the 907 utterances:

| Quality Dimension | Metric Value | Quality Gate | Status |
|---|:---:|:---:|:---:|
| **Slot-Level Precision** | **98.03%** | $\ge 98.0\%$ | ✅ PASS |
| **Slot-Level Recall** | **99.80%** | $\ge 98.0\%$ | ✅ PASS |
| **Slot-Level F1-Score** | **98.91%** | $\ge 98.0\%$ | ✅ PASS |
| **Verbatim Narrative Corruption Rate** | **0.00%** (0 chars altered) | $0.00\%$ | ✅ PASS |
| **Invented Numbers (Hallucination)** | **0.00%** (0 fabricated numbers) | $0.00\%$ | ✅ PASS |
| **Negative Control False Replacements** | **0.00%** (0 / 52) | $0.00\%$ | ✅ PASS |

De-identified ASR encounters live under [`data/transcripts/asr/`](data/transcripts/asr/): 8 files, one visit each. Five are `deidentified_capture` (English high-noise refraction, cataract glare, binocular vision, and cycloplegic wet refraction, plus the French capture). Three are `asr_pattern_transfer` (Italian, Spanish, and German), which replay the same acuity, swallowed-punctuation, and homophone errors in that locale's number words and acuity scale. They are labeled as transfers, not native recordings. Failing spans from these encounters are promoted into the gold sets above. A real capture replaces names and dates of birth before commit; the loader requires `"deidentified": true` and rejects email addresses, phone numbers, and labeled record numbers (`MRN`, `NHS`, `SSN`, `codice fiscale`).

---

## Example Transformations

### Example 1: Routine Presbyopic Examination & FHIR Export

#### Spoken Speech-to-Text Input

```text
07:52:59: Patient Fatima, age forty-nine, presented for a comprehensive examination with a primary complaint of difficulty reading small print and needing to hold the phone farther away to see clearly. Distance vision remained acceptable. Relevant visual demands include alternating between laptop work meetings and reading-printed contracts. No acute flashes, curtain-like field loss, severe ocular pain, or sudden vision loss reported unless otherwise specified. Unaided distance.
07:53:01: Visual acuity measured 2020 right eye and 2020 left eye.
07:53:26: Dry objective refraction by autorefractor or retinoscopy. Measured OD plus 0.50 sphere -0.25 cylinder at 175° and OS plus 0.75 sphere -0.50 cylinder at 10°. Subjective manifest refraction refine to OD plus 0.50 sphere.
07:53:55: Minus OH .25 cylinder at 180° and OS plus 0.75 sphere minus OH .50 cylinder at 10°. Final spectacle. Prescription OD plus 0.50 sphere -0.25 cylinder at 180°. OS plus 0.75 sphere minus OH .50 cylinder at 10°. Add plus 1.75 OU.
07:54:19: Anterior segment examination. Lids and lashes clear OU. Conjunctiva quiet OU. Cornea clear OU. Anterior chamber deep and quiet OU. Iris normal pattern OU. Lens trace nuclear sclerosis OU.
07:54:43: Tonometry. Goldmann applanation measured IOP 14 mmHg right eye and 15 mmHg left eye at 07:54.
07:55:01: Dilated fundus examination performed using 1 drop of tropicamide 1% OU. Vitreous clear OU. Optic nerve cup to disc ratio 0.3 round sharp margins pink rim OU. Macula flat dry good foveal reflex OU. Vessels normal caliber and course OU. Periphery 360 degrees flat and intact with no tears holes or detachments OU.
```

#### Normalized EHR Output

```text
07:52:59: Patient Fatima, age forty-nine, presented for a comprehensive examination with a primary complaint of difficulty reading small print and needing to hold the phone farther away to see clearly. Distance vision remained acceptable. Relevant visual demands include alternating between laptop work meetings and reading-printed contracts. No acute flashes, curtain-like field loss, severe ocular pain, or sudden vision loss reported unless otherwise specified. Unaided distance.
07:53:01: Visual acuity measured VA OD 20/20 and VA OS 20/20.
07:53:26: Dry objective refraction by autorefractor or retinoscopy. Measured OD +0.50 -0.25 x 175 and OS +0.75 -0.50 x 010. Subjective manifest refraction refine to OD +0.50.
07:53:55: -0.25 x 180 and OS +0.75 -0.50 x 010. Final spectacle. Prescription OD +0.50 -0.25 x 180. OS +0.75 -0.50 x 010. Add +1.75 OU.
07:54:19: Anterior segment examination. Lids and lashes clear OU. Conjunctiva quiet OU. Cornea: clear OU. AC: D&Q OU, no c/f. Iris normal pattern OU. Lens: trace NS OU.
07:54:43: Tonometry. Goldmann applanation measured IOP: 14 OD, 15 OS mmHg.
07:55:01: DFE performed using 1 drop of tropicamide 1% OU. Vitreous clear OU. Optic nerve C/D 0.3 OU round sharp margins pink rim OU. Macula flat dry good foveal reflex OU. Vessels normal caliber and course OU. Periphery 360 degrees flat and intact with no tears holes or detachments OU.
```

#### FHIR R4 Discrete Observations Generated

```json
{
  "resourceType": "Observation",
  "status": "final",
  "code": {
    "coding": [
      {
        "system": "http://loinc.org",
        "code": "28634-4",
        "display": "Refraction"
      }
    ],
    "text": "Subjective Refraction"
  },
  "subject": { "reference": "Patient/fatima-001" },
  "bodySite": {
    "coding": [
      {
        "system": "http://snomed.info/sct",
        "code": "24028007",
        "display": "Right eye structure"
      }
    ],
    "text": "OD"
  },
  "component": [
    {
      "code": {
        "coding": [
          {
            "system": "http://loinc.org",
            "code": "28637-7",
            "display": "Sphere"
          }
        ]
      },
      "valueQuantity": {
        "value": 0.5,
        "unit": "diopter",
        "system": "http://unitsofmeasure.org",
        "code": "[diop]"
      }
    },
    {
      "code": {
        "coding": [
          {
            "system": "http://loinc.org",
            "code": "28638-5",
            "display": "Cylinder"
          }
        ]
      },
      "valueQuantity": {
        "value": -0.25,
        "unit": "diopter",
        "system": "http://unitsofmeasure.org",
        "code": "[diop]"
      }
    },
    {
      "code": {
        "coding": [
          { "system": "http://loinc.org", "code": "28639-3", "display": "Axis" }
        ]
      },
      "valueQuantity": {
        "value": 180,
        "unit": "degrees",
        "system": "http://unitsofmeasure.org",
        "code": "deg"
      }
    }
  ]
}
```

### Example 2: Comprehensive Diabetic Eye Exam & ASR Resilience

Demonstrates multi-modal examination workflow: uncorrected DVA, habitual correction, autorefraction, manifest refraction, slit lamp, applanation tonometry, fundus evaluation, and red-flag urgent care warnings. Ambient conversational narrative is preserved 100% verbatim.

#### Spoken Speech-to-Text Input

```text
09:42:03: The unaided distance visual acuity is 2100 in the right eye. Now cover the right eye. Unaided visual acuity is 2080 in the left eye.
09:42:37: With the current correction, right eye acuity is 2025 using -2.00 sphere -0.75 cylinder cylinder at 170°. Left eye acuity is 2025 using -1.75 sphere -0.50 CYL cylinder at 10°. Both eyes together are functional, but there is room for refinement.
09:43:12: The dry non cycloplegia auto refraction gives -2.25 sphere -0.75 cylinder cylinder at 100 sixty-eight degrees in the right eye and -2.00 sphere -0.50 cylinder at 12° in the left eye.
09:43:59: With the right eye, this is -2.00 sphere -0.75 cylinder cylinder at 170°. Does the line look sharp and comfortable? Yes, the letters are clearer and not overly small.
09:44:10: For the left eye, the manifest result is -1.75 sphere -0.50 cylinder. Cylinder at 10°. Better one or better 2.
09:44:32: Good. The binocular result is stable frenir. The AD is plus 2.00 diopters in both eyes.
09:45:02: Mild ocular surface dryness is present, corneas are clear, anterior chambers are deep, and there is trace nuclear sclerosis.
09:45:16: I'm checking the eye pressure now. You may feel a light touch. The pressure is 15mm of mercury in the right eye and 17 in the left eye.
09:45:44: There are scattered microanurisms in both posterior poles and a few small hemorrhages without neo-vascularization or clinically apparent macular edema. Optic nerve are healthy.
09:46:01: -0.75 cylinder 170° for the right eye and -1.75 sphere -0.50 cylinder at 10° for the left eye.
09:46:20: Frenier, the AD is plus 2.00 diopters in both eyes. The final prescription remains close to the habitual correction.
09:46:31: Seek urgent care for sudden pain, marked redness, flashes, many new floaters, a curtain in the vision, or sudden visual loss. Thank you for explaining everything.
```

#### Normalized EHR Output

```text
09:42:03: DVA OD 20/100 sc. Now cover the right eye. DVA OS 20/80 sc.
09:42:37: With the current correction, VA OD 20/25 using -2.00 -0.75 x 170. VA OS 20/25 using -1.75 -0.50 x 010. Both eyes together are functional, but there is room for refinement.
09:43:12: The dry non cycloplegia auto refraction gives OD -2.25 -0.75 x 168 and OS -2.00 -0.50 x 012.
09:43:59: OD -2.00 -0.75 x 170. Does the line look sharp and comfortable? Yes, the letters are clearer and not overly small.
09:44:10: OS -1.75 -0.50 x 010. Better one or better 2.
09:44:32: Good. The binocular result is stable frenir. Add +2.00 OU.
09:45:02: Mild ocular surface dryness is present, Cornea: clear OU, AC: deep, Lens: trace NS.
09:45:16: I'm checking the eye pressure now. You may feel a light touch. IOP: 15 OD, 17 OS mmHg.
09:45:44: There are scattered microanurisms in both posterior poles and a few small hemorrhages without neo-vascularization or clinically apparent macular edema. ONH: healthy OU.
09:46:01: OD -0.75 x 170 and OS -1.75 -0.50 x 010.
09:46:20: Add +2.00 OU. The final prescription remains close to the habitual correction.
09:46:31: Seek urgent care for sudden pain, marked redness, flashes, many new floaters, a curtain in the vision, or sudden visual loss. Thank you for explaining everything.
```

### Example 3: Glare, Cataract & Presbyopia Examination

Demonstrates complete ambient encounter with visual complaints (night glare, haze), unaided DVA (`2060` $\rightarrow$ `DVA OD 20/60 sc`), habitual prescription check, ASR dysfluent autorefraction (`"odor refraction"`, `"sphere sphere"`), subjective manifest refinement, near add (`"frenier. The AD is plus 2.25 diopters"` $\rightarrow$ `Add +2.25 OU`), non-contact "brief puff" tonometry (`18mm of mercury right eye and 19 left eye` $\rightarrow$ `IOP: 18 OD, 19 OS mmHg`), lens nuclear sclerosis notation, dilated posterior evaluation, and progressive spectacle counseling.

#### Spoken Speech-to-Text Input

```text
04:32:55: The unaided distance visual acuity is 2060 in the right eye. Now cover the right eye. Unaided visual acuity is 2050 in the left eye. Now hold this near card at your normal reading distance.
04:33:37: With the current correction, right eye acuity is 2030 using -1.75 sphere, left eye acuity is 2025 using -1.50 sphere, sphere -0.50 cylinder at 170°. Both eyes together are functional, but there is room for refinement.
04:34:01: The dry non cycloplegics odor refraction gives -2.00 sphere sphere -0.50 cylinder at 20° in the right eye and -1.75 sphere -0.75 cylinder at 165° in the left eye.
04:34:42: With the right eye, this is -1.75 sphere -0.50 cylinder at 20°. Does the line look sharp and comfortable?
04:34:56: For the left eye, the manifest result is -1.50 sphere -0.75 cylinder at 165°.
04:35:21: Good. The binocular result is stable frenier. The AD is plus 2.25 diopters in both eyes.
04:35:51: Lids and ocular surface are quiet, Cornea's are clear. There is nuclear sclerosis grade II in both lenses, slightly greater in the right eye with mild cortical spokes.
04:36:07: You may feel a brief puff. The pressure is 18mm of mercury in the right eye and 19 in the left eye.
04:36:38: Vitreous is clear, optic nerve's are healthy with cup-to-disk ratio 0.35, Maculae are flat, vessels are age-appropriate, and the peripheral retina is attached.
04:37:02: We have completed the measurements. The final spectacle prescription is -1.75 sphere -0.50 cylinder cylinder at 20° for the right eye and -1.50 sphere -0.75 cylinder at 165° for the left eye. For near the ad is plus 2.25 diopters in both eyes.
04:37:23: I recommend progressive lenses with anti-reflective coating. The cataracts are currently mild to moderate.
```

#### Normalized EHR Output

```text
04:32:55: DVA OD 20/60 sc. Now cover the right eye. DVA OS 20/50 sc. Now hold this near card at your normal reading distance.
04:33:37: With the current correction, VA OD 20/30 using -1.75 sphere, VA OS 20/25 using -1.50 -0.50 x 170. Both eyes together are functional, but there is room for refinement.
04:34:01: The dry non cycloplegics odor refraction gives OD -2.00 -0.50 x 020 and OS -1.75 -0.75 x 165.
04:34:42: OD -1.75 -0.50 x 020. Does the line look sharp and comfortable?
04:34:56: OS -1.50 -0.75 x 165.
04:35:21: Good. The binocular result is stable frenier. Add +2.25 OU.
04:35:51: Lids and ocular surface are quiet, Cornea: clear OU. There is nuclear sclerosis grade II in both lenses, slightly greater in the right eye with mild cortical spokes.
04:36:07: You may feel a brief puff. IOP: 18 OD, 19 OS mmHg.
04:36:38: Vitreous is clear, ONH: healthy OU with cup-to-disk ratio 0.35, Maculae are flat, vessels are age-appropriate, and the peripheral retina is attached.
04:37:02: We have completed the measurements. The final spectacle prescription is OD -1.75 -0.50 x 020 and OS -1.50 -0.75 x 165. Add +2.25 OU.
04:37:23: I recommend progressive lenses with anti-reflective coating. The cataracts are currently mild to moderate.
```

### Example 4: Glaucoma Suspect Evaluation & Symmetrical Tonometry Synthesis

Demonstrates complete glaucoma suspect workup with family history, gonioscopy angle evaluation, optic disc cupping assessment (C/D 0.65 OD / 0.55 OS), natural language tonometry normalization with spelled-out units (`"twenty-one millimeters of mercury in the right eye and 21 in the left eye"` $\rightarrow$ `IOP: 21 OU mmHg`), hyperopic astigmatism with dysfluent cylinder, and presbyopic add.

#### Spoken Speech-to-Text Input

```text
04:31:04: I came for a routine exam but my father had glaucoma and I am worried about pressure.
04:32:35: The unaided distance visual acuity is 2040 in the right eye. Now cover the right eye. Unaided visual acuity is 2040 in the left eye.
04:33:09: Left eye acuity is 2020 using plus 1.50 sphere -0.50 cylinder at 85°.
04:33:40: For refraction or refraction gives plus 2.00 sphere, sphere, sphere -1.00 cylinder at 92° in the right eye and plus 1.75 sphere -0.75 cylinder at 88° in the left eye.
04:34:25: With a right eye, this is plus 1.75 sphere -0.75 cylinder at 90°. Does the line look sharp and comfortable?
04:34:40: For the left eye, the manifest result is plus 1.50 sphere -0.75 cylinder at 85°.
04:35:01: Good. The binocular result is stable frenir. The AD is plus 2.50 diopters in both eyes.
04:35:27: The angles are open in all quadrants with trabecular meshwork visible and mild pigmentation. There are no peripheral anterior synechii.
04:35:46: Cornea's are clear.
04:36:10: You may feel a light touch. The pressure is twenty-one millimeters of mercury in the right eye and 21 in the left eye.
04:36:51: The right optic disc is moderately cupped with cup-to-disk ratio 0.65 and possible inferior rim thinning. The left cup-to-disc ratio is 0.55 with a healthier rim. The final spectacle prescription is plus 1.75 sphere -0.75 cylinder at 90°.
04:36:56: For the right eye and plus 1.50 sphere sphere sphere.
04:37:00: -0.75 cylinder Cylinder at 85° for the left eye.
04:37:17: For near the AD is plus 2.50 diopters in both eyes. I recommend baseline Oct of the optic nerve's pachymetry and automated visual fields with pressure review in one month.
```

#### Normalized EHR Output

```text
04:31:04: I came for a routine exam but my father had glaucoma and I am worried about pressure.
04:32:35: DVA OD 20/40 sc. Now cover the right eye. DVA OS 20/40 sc.
04:33:09: VA OS 20/20 using +1.50 -0.50 x 085.
04:33:40: For refraction or refraction gives OD +2.00 -1.00 x 092 and OS +1.75 -0.75 x 088.
04:34:25: With a OD +1.75 -0.75 x 090. Does the line look sharp and comfortable?
04:34:40: OS +1.50 -0.75 x 085.
04:35:01: Good. The binocular result is stable frenir. Add +2.50 OU.
04:35:27: The angles are open in all quadrants with trabecular meshwork visible and mild pigmentation. There are no peripheral anterior synechii.
04:35:46: Cornea: clear OU.
04:36:10: You may feel a light touch. IOP: 21 OU mmHg.
04:36:51: The right optic disc is moderately cupped with cup-to-disk ratio 0.65 and possible inferior rim thinning. The left cup-to-disc ratio is 0.55 with a healthier rim. The final spectacle prescription is +1.75 -0.75 x 090.
04:36:56: For the right eye and +1.50 sphere.
04:37:00: OS -0.75 x 085.
04:37:17: Add +2.50 OU. I recommend baseline Oct of the optic nerve's pachymetry and automated visual fields with pressure review in one month.
```

---

## Latest Key Improvements & Technical Highlights

- **Dictated exam frames**: _"I focus now on the manifest refraction OD is 1 to 15 and 40"_ keeps the exam name and becomes `OD +1.00 -2.15 x 040`. The same frame writes visual acuity (`20 to 25` or `20 slash 25` → `VA OD 20/25`), intraocular pressure (`IOP: OD 14 mmHg`), and corneal pachymetry (`CCT: 540 OD µm`). In a cylinder, _to_ / _too_ / _two_ mean two. In a Snellen fraction, _to_, _slash_, and _forward slash_ mean a slash.
- **Closed-class filler noise**: Every exam that already has a copula accepts up to two filler words on each side of it (`actually`, `actuellement`, `ancora`, `todavía`, `noch`, and the rest of that closed list). A bare eye word may follow the exam name (`manifest refraction right is`), and `and` / `et` / `e` / `y` / `und` may join two numbers the exam already reads as a pair (`20 and 20`). Fixed phrases allow one of those filler words between tokens. Open-class words still stay narrative, so a blood-pressure sentence is not promoted to a finding.
- **Qualitative phrase coverage**: English, French, Italian, Spanish, and German fixed-phrase lexicons now claim stock slit-lamp and fundus wording that previously flushed as narrative (`Macula: flat, dry OU`, `Retina: flat, intact OU`, `Vitreous: clear OU`, `Conjunctiva: quiet OU`, `Iris: normal pattern OU`, `DFE: normal`, `Lids/Lashes: clear OU`), including one mild ASR slip per family.
- **Acoustic & Phonetic ASR Repair Engine (`acoustic_repair.py`)**:
  - Synthesizes phonetic decimals (e.g. `"minus OH .25 cylinder"` $\rightarrow$ `-0.25 cylinder`), collapses spoken degree symbols (`"at 180°"` $\rightarrow$ `axis 180`), repairs 4-digit concatenated Snellen numbers (`"2020"` $\rightarrow$ `20/20`, `"2400"` $\rightarrow$ `20/400`), and canonicalizes prepended prepositional laterality phrases (`"for the right eye"` $\rightarrow$ `OD`, `"for the left eye"` $\rightarrow$ `OS`).
- **Refraction Token & Diopter Shorthand Normalizer (`refraction_repair.py`)**:
  - Normalizes decimal-less rapid diopter sequence tokens (e.g. `"-1 2 40"` $\rightarrow$ `-1.00 -2.00 x 040`, `"1 1 and 40"` $\rightarrow$ `+1.00 -1.00 x 040`, `"1 1 and 1"` $\rightarrow$ `+1.00 -1.00 x 001`, `"125 075 8"` $\rightarrow$ `-1.25 -0.75 x 008`), unpacks joined cylinder-axis compound tokens (e.g. `"-0.75x180"` $\rightarrow$ `-0.75 x 180`), and standardizes habitual glasses measurement dictation (`"Glasses measured minus 125 minus 075 axis 8"` $\rightarrow$ `OD -1.25 -0.75 x 008`).
- **Multi-Sentence Contralateral Continuity & Clinical Discourse Robustness**:
  - Tonometry grammar (`iop.py`) tracks contralateral pressure findings across intervening clinical observations and narrative sentences (`"Right eye pressure 14. Cornea clear. Left eye 15"` $\rightarrow$ `IOP: 14 OD, 15 OS mmHg`).
  - Tear Breakup Time (`cornea.py`) supports conversational copulas such as `"TBUT is around 5 seconds"` $\rightarrow$ `TBUT: 5s OU`.
  - Expanded slit lamp lexicon (`phrases.py`) with canonical findings (`"angles open"`, `"quiet anterior chamber"`, `"quiet deep anterior chamber"`).
- **Real-Time Streaming & Ambient Scribing Architecture (Dimension 2)**:
  - Added `StreamingNormalizerBuffer` and `OffsetTracker` in `opto_normalizer.streaming` for low-latency live ASR chunk ingestion.
  - Implemented symmetrical bilateral holdback across speech pauses (IOP, Pachymetry, TBUT) and decimal fraction lookahead guards.
  - Added production WebSocket (`/v1/normalize/stream`) and Server-Sent Events (`/v1/normalize/sse`) streaming endpoints.
  - Added CLI streaming mode (`cat transcript.txt | optonorm --stream`) and Makefile target (`make test-streaming`).
- **Word-Form Spoken Tonometry & Symmetrical Bilateral Synthesis**:
  - Full parsing for natural-language and spelled-out pressure units (`"twenty-one millimeters of mercury in the right eye and 21 in the left eye"` $\rightarrow$ `IOP: 21 OU mmHg`).
  - Symmetrical readings across contralateral eyes automatically aggregate to `OU` in shorthand, while exporting 2 discrete monocular HL7 FHIR R4 Observations with LOINC `55284-4` (`OD = 21 mmHg`, `OS = 21 mmHg`).
- **Refraction Dysfluency & Cylinder Stutter Resilience**:
  - Standalone cylinder clauses tolerate capitalization and repeated words from acoustic ASR transcripts (`"-0.75 cylinder Cylinder at 85° for the left eye"` $\rightarrow$ `OS -0.75 x 085`, `"-1.75 sphere -0.50 cylinder cylinder at 20°"` $\rightarrow$ `OD -1.75 -0.50 x 020`).
- **Zero-Loss Clinical Narrative Preservation**:
  - Diagnostic and counseling discussions (glaucoma suspect status, gonioscopy angle visibility, optic disc C/D ratios with rim thinning, cataract nuclear sclerosis grades, urgency warnings) remain 100% verbatim, preventing clinician liability or loss of clinical nuance.
- **End-to-End Real-World Test Coverage**:
  - Test suite expanded to **1,659 passing tests** covering real-world simulation across 40 canonical encounter transcripts (8 subspecialties across 5 languages), multi-user authentication, personal phrase training, and 155 adversarial noisy ASR benchmark items across English, French, Italian, Spanish, and German.
- **Core Engine Package Version**:
  - Standardized as package `optonorm==0.45.0` with dynamic path resolution across all harvesting, benchmarking, and streaming pipelines.

---

## Installation & Quickstart

### Prerequisites

- Python 3.10+ (Recommended: Python 3.13)
- [`uv`](https://github.com/astral-sh/uv) (fast Python package manager)
- Node.js 20+ & npm (for interactive web playground development)

### 1. Clone & Setup Environment

```bash
git clone https://github.com/your-org/optonorm.git
cd optonorm

# Create virtual environment and sync dependencies
uv sync

# (Optional) Install web playground frontend dependencies
make web-install
```

### Quick Developer Commands (`make`)

A comprehensive [Makefile](Makefile) is included to automate all quality assurance, testing, benchmarking, and serving tasks:

```bash
make help           # Display interactive menu with all available targets
make check          # Run full code quality pipeline (ruff check + format check)
make test           # Run complete test suite with pytest (1647 passed, 1 skipped)
make test-streaming # Run real-time streaming & ambient scribing test suite (59 tests)
make benchmark      # Run gold benchmark (accuracy, hallucinations, latency)
make dev            # Start development FastAPI server with auto-reload (:8000)
make web-install    # Install web playground dependencies (npm install in web/)
make web-dev        # Launch interactive React 19 web playground dev server (:3000)
make web-build      # Build production web playground bundle in web/dist/
make web-lint       # Run oxlint & TypeScript type-checking across web codebase
make normalize      # Test clinical normalization via CLI
make fhir-doc       # Test FHIR R4 consultation document bundle export
make docker-up      # Start multi-service stack (Web :3000 + API :8000) via Docker Compose
make clean          # Remove cache and build artifacts
make auth-init      # Create the authentication database tables
make auth-list-users
make auth-create-user USERNAME=dr_roberto ROLE=clinician NAME="Dr. Roberto Rossi"
make auth-deactivate-user USERNAME=dr_roberto
make auth-create-key NAME="EHR Ingest Pipeline" ROLE=ehr_service EXPIRES_DAYS=365
make auth-revoke-key KEY_ID=key_01
make docker-auth-list-users
make docker-auth-create-user USERNAME=optoroot ROLE=superuser NAME="Superuser"
```

### 2. Run the Normalizer CLI (`main.py` or `optonorm`)

Root `main.py` and the installed package console script `optonorm` provide identical capabilities:

#### A. Clinical Transcript Normalization & FHIR Export

```bash
# Normalize directly from command line arguments
uv run main.py "04:32:00: Unaided distance visual acuity is 2040 right eye and 2040 left eye."
# or using console script (optonorm) with explicit locale:
uv run optonorm "acuité visuelle dix dixièmes œil droit" --locale fr

# Print HL7 FHIR R4 export directly to terminal
uv run optonorm "visual acuity 20/20 right eye, IOP 14 mmHg left eye" --fhir

# Export complete Clinical Document Bundle (Composition + DiagnosticReport)
uv run optonorm "visual acuity 20/20 right eye, IOP 14 mmHg left eye" --fhir --fhir-format document

# Export standalone US Core DiagnosticReport (LOINC 18603-1)
uv run optonorm "OD -2.50 -0.75 x 180, IOP 16 both eyes" --fhir --fhir-format diagnostic_report

# Generate Epic Hyperspace / EHR dotphrase macro (.OPTOEXAM)
uv run optonorm "OD -2.50 -0.75 x 180, IOP 16 both eyes" --dotphrase

# Select refraction FHIR format: composite (LOINC 28618-7, default) or panel (LOINC 28634-4)
uv run optonorm "OD -2.50 -0.75 x 180" --fhir --refraction-style panel

# Normalize from a text file and save normalized text and FHIR JSON to disk
uv run main.py -i visit.txt -o normalized.txt --output-fhir bundle.json --fhir-format document

# Pipe transcript from another command or file
cat transcript.txt | uv run main.py

# Interactive terminal diff visualizer (comparing raw ASR speech with normalized clinical shorthand)
uv run optonorm "visual acuity 20 20 right eye, IOP 14 mmHg left eye" --diff

# Real-time streaming mode over pipe or live standard input
cat live_stream.txt | uv run optonorm --stream --locale en

# Run built-in clinical demonstration
uv run main.py --demo

# Run in interactive prompt mode
uv run main.py
```

#### B. User, Role & API Key Administration (`auth`)

```bash
# Initialize SQLite or PostgreSQL authentication and training tables
uv run main.py auth init-db

# Create a clinician or trainer user
uv run main.py auth create-user --username dr_smith --role clinician --name "Dr. Jane Smith, OD"

# List all registered users and roles
uv run main.py auth list-users

# Deactivate a user
uv run main.py auth deactivate-user <user_id>

# Provision a machine API key (opto_live_ / opto_test_)
uv run main.py auth create-key --name "EHR Scribe Service" --role ehr_service

# Revoke an API key
uv run main.py auth revoke-key <key_id>
```

#### C. Secure Community Inquiries & Data Retention (`contact`)

```bash
# Initialize contact inquiries table and indexes
uv run main.py contact init-db

# List recent community contact submissions
uv run main.py contact list

# Prune submissions older than retention threshold (default 90 days)
uv run main.py contact prune --days 90
```

### 3. Bootstrap the FastAPI REST API Server

```bash
# Start server with default host (0.0.0.0) and port (8000)
uv run main.py serve
# or
optonorm serve --host 127.0.0.1 --port 8000 --reload
```

Interactive OpenAPI documentation will be immediately accessible at:

- **Swagger UI**: `http://localhost:8000/docs`
- **ReDoc**: `http://localhost:8000/redoc`

### 4. Run the Test Suite

```bash
uv run pytest -v
```

`tests/test_latency_redos_gate.py` times the gonioscopy, perimetry, OCT, motility, stereopsis, confrontation, and topography patterns on a 1,000-word note and requires that scan to finish in under 5 ms. The same test rejects catastrophic backtracking: long runs of spaces and repeated near-matches must finish in under 25 ms.

### 5. Run the Gold Benchmark Evaluation & Slot-Level IE Metrics

```bash
# Run benchmark across all supported languages (EN, FR, IT, ES, DE - 907 utterances) with slot-level Precision/Recall/F1.
# Exits 0 only when utterance accuracy is 100% and the invented-number count is 0.
# An accuracy drop or any invented number exits 1 (this is the CI gold-benchmark gate).
uv run python scripts/benchmark_eval.py --locale all

# Or run for a specific locale
uv run python scripts/benchmark_eval.py --locale fr

# Run adversarial noisy ASR benchmark suite (155 utterances). Same exit gate as the gold set.
uv run python scripts/benchmark_eval.py --noisy

# Export comprehensive markdown report for CI/CD audit gates
uv run python scripts/benchmark_eval.py --export-report reports/benchmark_report.md
```

A passing full `pytest` run writes the test count, and a passing all-locale `make benchmark` writes the mean latency, into [`web/src/data/projectMetrics.json`](web/src/data/projectMetrics.json). The homepage hero and the shields at the top of this file read those two numbers.

---

## FastAPI REST API & Plug-and-Play Integration

OptoNorm is engineered for seamless consumption across healthcare software architectures.

### Mode A: Plug-and-Play `APIRouter` for External FastAPI Projects

If your organization already runs a FastAPI EHR backend, telehealth service, or AI scribe pipeline, you can mount OptoNorm routes directly into your existing application with zero microservice overhead:

```python
from fastapi import FastAPI
from opto_normalizer.api import get_optonorm_router, optonorm_router

app = FastAPI(title="Hospital EHR Backend", version="2.0.0")

# Mount with custom route prefix and OpenAPI documentation tags
app.include_router(
    get_optonorm_router(prefix="/clinical/optometry", tags=["Optometry NLP Normalizer"])
)
```

Now, your application immediately exposes `/clinical/optometry/normalize`, `/clinical/optometry/fhir`, `/clinical/optometry/locales`, and `/clinical/optometry/health`.

### Mode B: Standalone HTTP Microservice Endpoints

#### 1. `POST /v1/normalize`

Normalizes a spoken or written clinical transcript into shorthand with a granular audit trail and sub-millisecond latency.

```bash
curl -X POST http://localhost:8000/v1/normalize \
  -H "Content-Type: application/json" \
  -d '{
    "transcript": "Visual acuity measured 2020 right eye and 2020 left eye.",
    "locale": "en"
  }'
```

**Response (`200 OK`)**:

```json
{
  "original_text": "Visual acuity measured 2020 right eye and 2020 left eye.",
  "normalized_text": "Visual acuity measured VA OD 20/20 and VA OS 20/20.",
  "detected_language": "en",
  "convention": "international",
  "edits": [
    {
      "start": 23,
      "end": 37,
      "original_text": "2020 right eye",
      "replacement_text": "VA OD 20/20",
      "finding_type": "visual_acuity",
      "rule_or_model_id": "va_snellen_4digit_regex",
      "confidence": 0.98,
      "flagged_for_review": false,
      "review_reason": null,
      "slots": {
        "distance_numerator": 20,
        "distance_denominator": 20.0,
        "laterality": "OD"
      }
    }
  ],
  "edit_count": 2,
  "latency_ms": 0.42
}
```

#### 2. `POST /v1/fhir`

Transforms clinical transcripts into HL7 FHIR R4 resources. You can test this interactively in your browser via Swagger UI at **[http://localhost:8000/docs](http://localhost:8000/docs)**, or via `curl`:

##### Option A: Standard HL7 FHIR Transaction Bundle (Default)

Generates discrete `Observation` resources ready for EHR ingestion. Spoken ocular medications are exported as `MedicationStatement` resources with an RxNorm coding when one is pinned. Clinical Decision Support `Flag` resources are included by default (`include_alerts` defaults to `true`). Pass `"include_alerts": false` to omit the flags:

```bash
curl -X POST http://localhost:8000/v1/fhir \
  -H "Content-Type: application/json" \
  -d '{
    "transcript": "visual acuity twenty twenty right eye, IOP 14 mmHg left eye",
    "patient_id": "patient-123"
  }'
```

##### Option B: Complete Clinical Document Bundle (`format: "document"`)

Generates an enterprise document bundle (`type="document"`) with US Core `Composition` (LOINC `18603-1`), `DiagnosticReport`, and human-readable XHTML narrative:

```bash
curl -X POST http://localhost:8000/v1/fhir \
  -H "Content-Type: application/json" \
  -d '{
    "transcript": "visual acuity twenty twenty right eye, IOP 14 mmHg left eye, pupils equal round and reactive",
    "patient_id": "patient-123",
    "practitioner_id": "dr-smith",
    "format": "document"
  }'
```

##### Option C: Standalone US Core `DiagnosticReport` (`format: "diagnostic_report"`)

Returns LOINC `18603-1` (_Optometry Consult note_) with formatted XHTML table and observation references:

```bash
curl -X POST http://localhost:8000/v1/fhir \
  -H "Content-Type: application/json" \
  -d '{
    "transcript": "OD minus 2.50 minus 0.75 axis 180, IOP 16 both eyes",
    "patient_id": "patient-123",
    "format": "diagnostic_report"
  }'
```

##### Option D: Standalone US Core `Composition` (`format: "composition"`)

Organized into 5 clinical sections (_Visual Function_, _Refraction_, _IOP_, _Corneal Health_, _Biomicroscopy_):

```bash
curl -X POST http://localhost:8000/v1/fhir \
  -H "Content-Type: application/json" \
  -d '{
    "transcript": "acuité visuelle dix dixièmes œil droit, tension oculaire 14 millimètres de mercure œil droit",
    "locale": "fr",
    "format": "composition"
  }'
```

#### 3. `GET /v1/locales`

Lists supported language codes (`en`, `fr`, `it`, `es`, `de`), clinical scales, and default conventions. `POST /v1/normalize`, `POST /v1/fhir`, and `GET /v1/normalize/sse` return **422** when `locale` is not one of those codes or `auto`. A region tag such as `de-DE` resolves to `de`. An unknown `Accept-Language` tag is skipped so detection can still run. The WebSocket rejects an unknown `locale` query with close code 1008.

#### 4. `GET /v1/health`

Liveness probe reporting process status, version, loaded locale codes, and `database`. `database` is `ok` when the auth store opened or authentication is off. When the auth database refuses the connection, the process still stays up and `database` carries a short error that omits the connection string. A `200` does not prove the grammars loaded or that a sample note would normalize.

#### 5. `WebSocket /v1/normalize/stream`

Bi-directional low-latency WebSocket endpoint for real-time speech-to-text chunked streaming and ambient scribing. Ideal for direct integration with browser microphones, ASR providers (Whisper, Deepgram), or real-time clinical scribing UIs.

**Query Parameters**:
- `locale`: Optional language (`en`, `fr`, `it`, `es`, `de`, or `auto`).
- `convention`: Output shorthand convention (`international` or `localized`).
- `patient_id`: Patient identifier for discrete FHIR observation generation.

**Message Protocol (Client -> Server)**:
```json
{
  "chunk": "Visual acuity measured 2020 right eye",
  "is_final": true,
  "action": "feed"
}
```
Available actions:
- `feed`: Ingests an incoming audio/transcript chunk into the sliding window buffer.
- `flush`: Forces flushing of all held-back buffers and pending findings (e.g. at end of exam).
- `reset`: Clears the buffer and offset tracker for a new patient encounter.

**Event Protocol (Server -> Client)**:
```json
{
  "event_type": "finding_committed",
  "emitted_text": "VA OD 20/20",
  "original_text": "2020 right eye",
  "finding_type": "visual_acuity",
  "slots": {
    "distance_numerator": 20,
    "distance_denominator": 20.0,
    "laterality": "OD"
  },
  "fhir_observations": [],
  "flagged_for_review": false,
  "review_reason": null,
  "error_message": null
}
```

Event types:

- `finding_committed`: verified shorthand written into the note, with FHIR observations when the build succeeds.
- `review_required`: a guard rejected the proposal. `emitted_text` is the proposed shorthand, `original_text` is the speech that stays in the note, and `flagged_for_review` is true. No FHIR observation is emitted for that span.
- `error`: a committed finding's FHIR build failed. `error_message` explains the failure instead of returning a silent empty observation list.
- `narrative_flush`, `interim_update`, and `session_end` cover verbatim speech, unconfirmed ASR hypotheses, and end of encounter.

#### 6. `GET /v1/normalize/sse`

Server-Sent Events (SSE) endpoint providing unidirectional HTTP streaming of normalization events from continuous speech streams.

**Query Parameters**:
- `text`: URL-encoded transcript chunk. This value, and `patient_id` when supplied, are part of the request URL, so they are copied into access logs, proxy logs, and browser history.
- `locale`: Optional language (`en`, `fr`, `it`, `es`, `de`, `auto`).
- `convention`: `international` (default) or `localized`.
- `patient_id`: Patient identifier. Prefer a non-identifying token; the query string is not a private channel.

```bash
curl -N "http://localhost:8000/v1/normalize/sse?text=visual%20acuity%202020%20right%20eye.%20IOP%2014%20mmHg%20left%20eye."
```

#### 7. `GET /v1/contact/token` and `POST /v1/contact`

Public contact channel for community inquiries, ASR mishearing bug reports, new language locale proposals, and parser contributions. Fully decoupled from clinical auth credentials, allowing public contributor outreach while enforcing strict healthcare security controls.

- **Anti-Bot Cryptographic Time Ticket (`GET /v1/contact/token`)**: Issues a cryptographically signed HMAC-SHA256 token binding a UNIX timestamp and single-use random nonce. Submissions issued in $< 3.0$ seconds (automated bot scripts) or $> 7,200$ seconds (stale replays) are rejected with HTTP 400. Replayed nonces are rejected by an in-memory thread-safe `NonceCache`.
- **Honeypot Decoy (`contact_code`)**: Bot scrapers populating the hidden decoy input are silently accepted (returns HTTP 200 with a dummy submission reference) without writing to the database or triggering alerts.
- **Strict Zero-PHI Enforcement**: Server-side high-precision regex detectors inspect all fields for patient phone numbers, emails, and labeled healthcare identifiers (`MRN`, `NHS`, `SSN`, Italian Codice Fiscale). Genuine optometric notation (axes, diopters, keratometry) is preserved without false positives. Detected PHI triggers immediate rejection with HTTP 422.
- **Sliding-Window Rate Limiting**: In-memory sliding-window limiter allows a burst of up to 3 submissions per 60 seconds and 10 per hour per client IP. Client IPs are irreversibly hashed with salted SHA-256 (`OPTONORM_CONTACT_IP_SALT`) to protect user privacy.
- **Data Retention & Storage**: Standalone SQLite database (`data/contact.db` or PostgreSQL via `OPTONORM_CONTACT_DATABASE_URL`). When `OPTONORM_CONTACT_DATABASE_URL` is unset, it automatically inherits `OPTONORM_AUTH_DATABASE_URL` or `DATABASE_URL` if configured for Postgres. Submissions older than 90 days are automatically pruned on server startup.

```bash
# Initialize contact inquiry tables and indexes in SQLite or PostgreSQL:
uv run main.py contact init-db

# Or initialize both authentication and contact tables together:
uv run main.py auth init-db
```

```bash
# 1. Fetch anti-bot ticket
TICKET=$(curl -s http://localhost:8000/v1/contact/token)
TOKEN=$(echo "$TICKET" | jq -r .token)
NONCE=$(echo "$TICKET" | jq -r .nonce)
TS=$(echo "$TICKET" | jq -r .timestamp)

# 2. Wait 3 seconds to clear the anti-bot time gate, then submit
sleep 3
curl -X POST http://localhost:8000/v1/contact \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Dr. Jane Smith, OD",
    "email": "jane.smith@eyecare.org",
    "category": "clinical_vocabulary",
    "subject": "[Clinical Vocabulary]: Pediatric Fix & Follow",
    "message": "Proposal to recognize F&F and CSM across bilingual pediatric exams.",
    "phi_confirmed": true,
    "contact_code": "",
    "anti_bot_token": "'"$TOKEN"'",
    "anti_bot_nonce": "'"$NONCE"'",
    "token_timestamp": '"$TS"'
  }'
```

#### 8. Training & Vocabulary Endpoints (`/v1/training`)

Programmatic management for shared clinical phrases, personal doctor-specific rules, and live grammar vocabulary extensions.

- **Shared Training Rules (`POST /v1/training/rules`, `GET /v1/training/rules`)**: Requires `vocabulary:train` (roles: `admin`, `trainer`, `superuser`). Taught phrases are induced into regex slot rules, checked against the gold-standard benchmark to prevent regressions, and applied globally across all normalization requests.
- **Personal Clinician Phrases (`POST /v1/training/personal/rules`, `GET /v1/training/personal/rules`)**: Requires `vocabulary:personal` (roles: `clinician`, `technician_scribe`, `trainer`, `superuser`). Clinicians teach phrases scoped exclusively to their own user ID (`caller_context`). Personal rules take precedence over shared rules (Tier 1 > Tier 2) and skip global gold conflict checks.
- **Dry-Run Preview**: `POST /v1/training/preview` and `POST /v1/training/personal/preview` test rule induction and slot extraction on sample dictation before committing.
- **Rule Lifecycle**: Rules can be deactivated via `POST /v1/training/rules/{rule_id}/disable` (or `/personal/rules/{rule_id}/disable`) and permanently deleted via `DELETE /v1/training/rules/{rule_id}` (or `/personal/rules/{rule_id}`).
- **Vocabulary Literals (`POST /v1/training/vocabulary`, `GET /v1/training/vocabulary`, `DELETE /v1/training/vocabulary/{token_id}`)**: Requires `vocabulary:train`. Slices extra clinical literals (e.g. IOP markers, laterality tokens) into grammar templates after regex safety validation.
- **Rules Export (`GET /v1/training/export`)**: Requires `vocabulary:train`. Streams active rules in newline-delimited JSON (`application/x-ndjson`).

### Deployment limits

Authentication is on by default. `POST /v1/normalize`, `POST /v1/fhir`, `GET /v1/normalize/sse`, and the WebSocket at `/v1/normalize/stream` require a Bearer access token or an `X-API-Key` header. `GET /v1/health`, `GET /v1/locales`, `GET /v1/contact/token`, and `POST /v1/contact` stay public. Set `OPTONORM_AUTH_ENABLED=false` to restore the open gateway for a private lab or an existing unauthenticated pipeline.

| Control | Default | Tighten with |
| --- | --- | --- |
| Authentication | Required (`OPTONORM_AUTH_ENABLED=true`) | `OPTONORM_AUTH_ENABLED=false` restores the open gateway |
| Shared vocabulary training | Requires `vocabulary:train` scope | Scoped to `admin`, `trainer`, `superuser` |
| Personal training phrases | Requires `vocabulary:personal` scope | Scoped to authenticated user ID (`clinician`, `technician_scribe`, `trainer`, `superuser`) |
| JWT secret | Required when authentication is on, at least 32 bytes | `OPTONORM_JWT_SECRET` |
| Auth database | SQLite file `data/auth.db` | `OPTONORM_AUTH_DB_PATH` or `OPTONORM_AUTH_DATABASE_URL` (`postgresql://`, inherits `DATABASE_URL`) |
| Contact database | SQLite file `data/contact.db` | `OPTONORM_CONTACT_DB_PATH` or `OPTONORM_CONTACT_DATABASE_URL` (inherits `OPTONORM_AUTH_DATABASE_URL` / `DATABASE_URL`) |
| Contact IP salt | Derived from JWT secret or random | `OPTONORM_CONTACT_IP_SALT` |
| Contact notifications | Disabled (database only) | `OPTONORM_CONTACT_NOTIFICATION_EMAIL` (target recipient address) |
| Contact sender address | `no-reply@optonorm.com` | `OPTONORM_CONTACT_EMAIL_FROM` |
| Contact email delivery (API) | Disabled | `OPTONORM_RESEND_API_KEY` (transactional HTTPS REST via Resend) |
| Contact email delivery (SMTP)| Disabled | `OPTONORM_SMTP_HOST`, `OPTONORM_SMTP_PORT`, `OPTONORM_SMTP_USERNAME`, `OPTONORM_SMTP_PASSWORD`, `OPTONORM_SMTP_USE_TLS` |
| Contact webhook | Disabled (database only) | `OPTONORM_CONTACT_WEBHOOK_URL` (JSON POST dispatch) |
| Browser origins | Any `Origin` is reflected, and `Access-Control-Allow-Credentials` is `true` | `OPTONORM_CORS_ORIGINS` |
| Transcript length | No character cap | `OPTONORM_MAX_TRANSCRIPT_LENGTH` |

- **Configuration.** Copy [`.env.example`](.env.example) to `.env` for local commands such as `make dev` and `make auth-init`. Process environment variables override that file. The Docker image does not contain `.env`. Compose, Kubernetes, and systemd should inject the same `OPTONORM_*` names from a secret manager. `OPTONORM_SECRETS_DIR` can point at a directory of secret files named `jwt_secret`, `bootstrap_admin_password`, and `auth_database_url`; environment variables override those files.
- **Sessions.** `POST /v1/auth/login` checks an Argon2 password hash and returns a 15-minute access JWT. The 7-day refresh token is an `HttpOnly` `SameSite=Lax` cookie named `optonorm_refresh` (path `/v1/auth`). Set `OPTONORM_AUTH_COOKIE_SECURE=true` on HTTPS. `POST /v1/auth/refresh` rotates the cookie. Reuse of a rotated refresh token revokes that user's refresh tokens.
- **API keys.** `optonorm auth create-key` prints an `opto_live_` or `opto_test_` secret once. Only the SHA-256 hash is stored. Send it as `X-API-Key`. Revocation takes effect on the next request.
- **Roles.** `superuser` has every permission, including streaming and document signing. `admin` manages users, keys, and configuration, can normalize, export FHIR, and train global vocabulary (`vocabulary:train`), but cannot stream or sign. `trainer` normalizes, streams, exports FHIR, and trains shared and personal vocabulary (`vocabulary:train`, `vocabulary:personal`), but cannot sign documents. `clinician` normalizes, streams, exports, signs documents, teaches personal phrases (`vocabulary:personal`), and reads their own audit events. `technician_scribe` normalizes, streams, exports, and teaches personal phrases (`vocabulary:personal`), but cannot sign. `auditor_billing` exports FHIR and reads the audit log, but cannot normalize or stream. `ehr_service` is the machine role; `--allow-sign` adds document signing to that key only.
- **Signed FHIR.** `document`, `composition`, and `diagnostic_report` require `fhir:sign`. The author is the caller's `practitioner_id`, and the resource contains a FHIR R4 `Provenance` entry. Transaction and collection bundles require `fhir:export`.
- **Streams.** Browsers obtain a 60-second single-use ticket from `POST /v1/auth/stream-ticket` and connect to `/v1/normalize/stream?ticket=...`. Non-browser clients may send `Authorization: Bearer`, `X-API-Key`, or `Sec-WebSocket-Protocol: bearer, <token>`. A missing or insufficient credential closes the socket with `1008`.
- **Audit.** Authenticated clinical requests append an audit row with the actor, role, action, client IP, and a SHA-256 of the transcript. The transcript text is not stored. Clinicians see only their own rows at `GET /v1/auth/audit`. Admins and auditors see every row. Ambient-stream inserts run on a worker thread, so a slow or failed write does not delay `reconcile` or close the socket. The log line is the error text with any Postgres URL removed.
- **Postgres.** `OPTONORM_AUTH_DATABASE_URL` connects with a 5-second timeout and TCP keepalives. A closed connection is replaced on the next query.
- **First administrator.** `uv run main.py auth create-user --username admin --role admin --name "System Administrator"` prompts for a password. If `OPTONORM_BOOTSTRAP_ADMIN_PASSWORD` is set and the database has no users, startup creates username `admin`. No default password is built into the image.
- **CORS.** [`src/opto_normalizer/api/app.py`](src/opto_normalizer/api/app.py) mounts `CORSMiddleware` with `allow_credentials=True`. When `OPTONORM_CORS_ORIGINS` is unset, the allowlist is `*`, and Starlette reflects the request `Origin` on every response. Set a comma-separated allowlist before the port is reachable from a browser you do not control, for example `OPTONORM_CORS_ORIGINS=https://ehr.example,https://scribe.example`.
- **Transcript cap.** When `OPTONORM_MAX_TRANSCRIPT_LENGTH` is a positive integer, `POST /v1/normalize`, `POST /v1/fhir`, the SSE `text` query, and each WebSocket message are rejected once the character count (including text already held in the streaming buffer) exceeds it. The status is **413**. A non-numeric value fails process startup. The check runs after the body has been read, so a reverse proxy should still set its own body-size limit.
- **SSE query string.** The transcript and `patient_id` travel in the URL. Do not put real identifiers there.
- **Event loop.** `POST /v1/normalize` and `POST /v1/fhir` are synchronous routes, so FastAPI runs them in a worker thread. The WebSocket and SSE handlers call `normalize()` on the asyncio event loop. A long streaming transcript stalls other connections on that worker. Stream audit inserts do not: they run in a worker thread and the next socket message, including `flush`, is read without waiting for the commit.
- **Health.** `GET /v1/health` reports that the process is up. The `database` field is `ok`, or a short auth-database error when the store failed to open. The connection string is not included. A refused database does not stop the process. The playground navbar requests it once per page load for the status badge.

```bash
OPTONORM_JWT_SECRET="$(openssl rand -hex 32)" \
OPTONORM_CORS_ORIGINS=https://ehr.example \
OPTONORM_MAX_TRANSCRIPT_LENGTH=20000 \
uv run main.py serve
```

```bash
uv run main.py auth init-db
uv run main.py auth create-user --username dr_roberto --role clinician --name "Dr. Roberto Rossi" --practitioner-id OD-88421
uv run main.py auth create-key --name "EHR Ingest Pipeline" --role ehr_service --expires-days 365
uv run main.py auth list-users
uv run main.py auth revoke-key --key-id key_01
```

The same commands are available as Make targets. Omit `PASSWORD` and Make prompts for it. Set `ALLOW_SIGN=1` to grant `fhir:sign` on an `ehr_service` key, or `TEST_KEY=1` to issue an `opto_test_` key. `make auth-*` writes to the host database `data/auth.db`. `make docker-auth-*` runs the same commands in the Compose API container, which uses the `optonorm-auth` volume. Sign in through http://localhost:3000 with a user created by `docker-auth-*`.

```bash
make auth-init
make auth-create-user USERNAME=dr_roberto ROLE=clinician NAME="Dr. Roberto Rossi" PRACTITIONER_ID=OD-88421
make auth-create-key NAME="EHR Ingest Pipeline" ROLE=ehr_service EXPIRES_DAYS=365
make auth-list-users
make auth-deactivate-user USERNAME=dr_roberto
make auth-revoke-key KEY_ID=key_01
make docker-auth-create-user USERNAME=optoroot ROLE=superuser NAME="Superuser"
make docker-auth-list-users
```

---

## Interactive Clinical Web Playground (React 19 & Redux Toolkit)

OptoNorm includes a modern, production-grade clinical web playground located in [`web/`](web/) engineered for clinicians, EHR integrators, and clinical NLP researchers. Built on **React 19**, **Redux Toolkit**, **TanStack Form**, and **TanStack Query**, it provides a real-time cockpit for transcript normalization, live ASR streaming simulation, and discrete FHIR R4 inspection.

```
┌────────────────────────────────────────────────────────────────────────┐
│  OptoNorm Clinical Playground Cockpit                                  │
│  [Logo] OptoNorm v0.45.0   ● API Online (0.56 ms)     [GitHub] [Theme] │
├───────────────────────────────────┬────────────────────────────────────┤
│  Clinical Input & Presets         │  Live Scribe & Output Inspector    │
│  - Patient & Practitioner ID      │  - Tabs: Visual Diff | Findings    │
│  - Locale & Convention Selectors  │    | FHIR JSON | Consultation Note │
│  - 10 Multilingual Presets        │  - Redux Ambient Scribe Simulator  │
│  - Batch Normalization Mutation   │  - WebSocket Stream (/v1/.../ws)   │
│  - Live Character Count           │  - Holdback Buffer Visual HUD      │
└───────────────────────────────────┴────────────────────────────────────┘
```

### Key Modules & Capabilities

1. **Clinical Input Form & Multilingual Presets (`@tanstack/react-form`)**:
   - Zero-lag typed input using fine-grained reactive subscriptions (`form.Subscribe`).
   - Configurable encounter metadata: Patient ID, Practitioner ID, Locale (`auto`, `en`, `fr`, `it`, `es`, `de`), Shorthand Convention (`international`, `localized`), and FHIR Bundle Format (`transaction`, `collection`, `document`, `diagnostic_report`).
   - One-click loader for 10 clinical presets spanning Presbyopia Refraction, Cataract Evaluation (LOCS III), Glaucoma Suspect (diurnal IOP & pachymetry), Dry Eye Disease (TBUT & stain grading), and Pediatric Strabismus.

2. **Redux Ambient Scribe Simulator (`StreamingSimulator.tsx`)**:
   - Powered by a centralized Redux Toolkit state machine ([`streamingSlice.ts`](web/src/store/slices/streamingSlice.ts)) fully inspectable in Redux DevTools.
   - Connects over full-duplex WebSockets to `/v1/normalize/stream`, streaming simulated speech tokens at customizable playback rates ($0.5\times$ to $5.0\times$).
   - **Holdback Buffer HUD**: Real-time visualization of unilateral findings held in the buffer across clinician pauses, demonstrating bilateral symmetry synthesis before committing canonical `OU` shorthand.
   - **Review flags**: A `review_required` event keeps the spoken span in the note, shows the proposed shorthand as flagged in the event feed and clinical diff, and does not write that proposal into the copied shorthand. A FHIR build failure surfaces as an `error` event and an error banner.

3. **Visual Clinical Diff Viewer (`ClinicalDiffViewer.tsx`)**:
   - Dual visualization modes: **Inline** and **Side-by-Side**.
   - Color-coded tokens: Strikethrough red for replaced conversational speech, bold green for standardized optometric shorthand, amber for flagged clinical review warnings, and untouched narrative preserved 100% verbatim.

4. **Structured Findings Table (`FindingsTable.tsx`)**:
   - Categorized summary of all parsed findings with laterality, extracted value, and direct mapping to verified healthcare standards:
     - **LOINC** measurement and panel codes (`8629-0`, `28634-4`, `55284-4`, `70949-3`, etc.).
     - **SNOMED CT** anatomical and qualitative concepts (`28400003`, `414646002`, `231872005`, etc.).

5. **Interactive HL7 FHIR R4 Inspector (`FhirInspector.tsx`)**:
   - Collapsible JSON resource tree for discrete `Observation`, `DiagnosticReport`, and `Composition` bundles.
   - Fast filtering by clinical category (`va`, `refraction`, `iop`, `exam`).
   - One-click copy to clipboard and `.json` file download.

6. **US Core 6.1.0 Clinical Consultation Document View (`ClinicalDocumentView.tsx`)**:
   - Human-readable XHTML clinical note generated directly from FHIR `Composition.text`.
   - Complete with patient/practitioner header, structured exam sections, one-click export (JSON / HTML), and dedicated print styling (`@media print`).

### Launching the Web Playground

#### Mode 1: Single-Command Docker Compose (Full Stack)

Launch both the FastAPI backend and the React 19 web frontend with a single command:

```bash
# Build and launch both containers in the background
docker compose up --build -d

# Check health and view logs
make docker-logs
```

- **Web Playground UI**: Accessible at [`http://localhost:3000`](http://localhost:3000)
- **FastAPI REST & WebSocket Server**: Accessible at [`http://localhost:8000`](http://localhost:8000)
- **Interactive Swagger Docs**: Accessible at [`http://localhost:8000/docs`](http://localhost:8000/docs)

To stop services:
```bash
docker compose down
```

#### Mode 2: Local Dual-Server Development

For rapid local frontend or backend iteration with hot module replacement (HMR):

```bash
# 1. Install frontend dependencies
make web-install

# 2. Copy local settings and set OPTONORM_JWT_SECRET to at least 32 bytes
#    (openssl rand -hex 32). Uncomment OPTONORM_BOOTSTRAP_ADMIN_PASSWORD
#    to create username admin on an empty database.
cp .env.example .env

# 3. Start FastAPI backend (in Terminal 1).
# Authentication is on by default and refuses to start without that secret.
make dev

# Open gateway for a private lab that does not sign in:
# OPTONORM_AUTH_ENABLED=false make dev

# 4. Start Vite React 19 dev server with proxy to :8000 (in Terminal 2)
make web-dev
```

- Vite automatically proxies `/v1` HTTP requests and `/v1/normalize/stream` WebSocket traffic to the local FastAPI server at `http://127.0.0.1:8000`.

---

## Containerized Deployment (Docker & Compose)

OptoNorm provides multi-stage unprivileged Docker containers for secure enterprise deployment:

- **API Container ([`Dockerfile`](Dockerfile))**: Built on `python:3.13-slim` using `uv`, running as non-root user `optonorm` (UID 10001) on port `8000`. The image installs runtime dependencies only (`uv sync --no-dev`), so pytest is not in the container. [`docker/entrypoint.sh`](docker/entrypoint.sh) prepares `/app/auth` and drops privileges. Compose interpolates `OPTONORM_JWT_SECRET` from the project `.env` when that variable is set, and otherwise uses a local-only fallback. The image does not contain `.env`. Replace the fallback before any shared deployment. The `optonorm-auth` volume is mounted at `OPTONORM_AUTH_DB_PATH`. Set `OPTONORM_AUTH_ENABLED=false` to restore the open gateway. `OPTONORM_CORS_ORIGINS` and `OPTONORM_MAX_TRANSCRIPT_LENGTH` are described under [Deployment limits](#deployment-limits).
- **Web Container ([`web/Dockerfile`](web/Dockerfile))**: Multi-stage build (`node:22-alpine` builder, `nginxinc/nginx-unprivileged:alpine` runner) serving the compiled React 19 SPA on unprivileged port `8080` (mapped to host port `3000`). Reverse proxies API and WebSocket requests with healthcheck dependency on `optonorm-api`.

```bash
# Start multi-service stack with Docker Compose
docker compose up --build -d

# Verify API health
curl http://localhost:8000/v1/health

# Verify Web playground
curl http://localhost:3000/

# Stream logs
docker compose logs -f

# Teardown
docker compose down
```

### Isolated Cloudfare Stack (Unified Container + PostgreSQL + pgAdmin)

For Cloudflare Containers, Cloudflare Tunnel, or containerized production deployment, an isolated all-in-one stack is available:

- **Unified Container ([`DockerfileCloudfare`](DockerfileCloudfare))**: Multi-stage image bundling Vite/React 19 SPA (`/usr/share/nginx/html`) and FastAPI with an in-container Nginx reverse proxy on front-facing port `8000`.
- **Dedicated Compose ([`docker-compose.cloudfare.yml`](docker-compose.cloudfare.yml))**: Orchestrates `optonorm-app` on port `8000`, `optonorm-db` (PostgreSQL 17) on port `5432`, and `optonorm-pgadmin` (pgAdmin 4) on port `5050`.
- **Database Awareness**: Configured with `OPTONORM_AUTH_DATABASE_URL=postgresql://${OPTONORM_POSTGRES_USER:-optonorm}:${OPTONORM_POSTGRES_PASSWORD:-optonorm_secret}@optonorm-db:5432/${OPTONORM_POSTGRES_DB:-optonorm}`. Automatically provisions auth and audit tables on boot.

```bash
# Build unified Cloudfare image
make cloudfare-build

# Start isolated stack in background (app, PostgreSQL, pgAdmin)
make cloudfare-up

# Verify unified frontend & API
curl http://localhost:8000/           # React 19 SPA
curl http://localhost:8000/v1/health  # FastAPI health

# Access pgAdmin 4 panel
open http://localhost:5050/           # Login: admin@example.com / admin1234

# Stream logs & teardown
make cloudfare-logs
make cloudfare-down
```

#### Production Deploy to Cloudflare (Wrangler + Workers Containers)

1. Authenticate with Cloudflare:
   ```bash
   npx wrangler login
   ```
2. Configure production secrets in Cloudflare:
   ```bash
   cd cloudfare
   npx wrangler secret put OPTONORM_JWT_SECRET
   npx wrangler secret put OPTONORM_AUTH_DATABASE_URL
   npx wrangler secret put OPTONORM_BOOTSTRAP_ADMIN_PASSWORD
   ```
3. Deploy frontend assets & backend container to `https://optonorm.com`:
   ```bash
   make deploy
   ```

#### Production Performance Architecture

To maintain sub-2ms response latencies and eliminate cold-start/network-hop friction across the Cloudflare Worker $\rightarrow$ Container $\rightarrow$ Neon Postgres topology:

- **Container-Tuned Argon2**: Configured with OWASP-aligned parameters (`time_cost=2, memory_cost=19456, parallelism=1`) and lazy dummy hash initialization, avoiding CPU throttling on container vCPUs and preventing startup latency spikes. A successful login rewrites a stored hash that still uses older parameters, so the following sign-in uses this cost.
- **Single-Transaction Token Rotation**: `rotate_refresh_token` loads the refresh token and user together, then revokes the old token and inserts the new one in that same transaction. Paths that do not rotate roll the transaction back. The revoke matches only a token that is still active, so a concurrent refresh is treated as reuse.
- **In-Memory Principal TTL Cache**: 60-second thread-safe user record cache (`get_cached_user`) in the container process, eliminating redundant database round trips on consecutive authenticated `/v1` button clicks. An invalidation that arrives while a lookup is reading the database is kept, and that lookup does not write the user back into the cache.
- **Non-Blocking Background Auditing**: Append-only audit records on `/v1/normalize`, `/v1/fhir`, and `/v1/normalize/sse` run via `BackgroundTasks` without blocking HTTP response delivery to the client.
- **Optimistic Header Rendering**: The React SPA caches `optonorm_auth_required` in `localStorage`, rendering header navigation controls on frame 1 without waiting for network session probing.
- **Container Instance Sizing**: Dedicated `standard-1` (1/2 vCPU, 4 GiB memory) configuration in `wrangler.jsonc` ensuring ample compute for concurrent normalization and streaming.

---

## Python In-Process API Usage

```python
from opto_normalizer import normalize, export_to_fhir_bundle
from opto_normalizer.i18n import Locale, ShorthandConvention

transcript = (
    "Visual acuity measured 2020 right eye and 2020 left eye. "
    "Refraction OD plus 0.50 sphere -0.25 cylinder at 180°. "
    "Tonometry measured IOP 14 mmHg OD and 15 mmHg OS."
)

# 1. Normalize text (automatic language detection or explicit locale)
result = normalize(
    transcript,
    locale=Locale.AUTO,  # or Locale.EN, Locale.FR, Locale.IT, Locale.ES
    convention=ShorthandConvention.INTERNATIONAL,
)

print(result.normalized_text)
# Output:
# Visual acuity measured VA OD 20/20 and VA OS 20/20. Refraction OD +0.50 -0.25 x 180. Tonometry measured IOP: 14 OD, 15 OS mmHg.

# 2. Inspect reversible edit audit trail
for edit in result.edits:
    print(
        f"[{edit.finding_type.value}] '{edit.original_text}' -> '{edit.replacement_text}' (flagged={edit.flagged_for_review})"
    )

# 3. Export to HL7 FHIR R4 Transaction Bundle or Document Bundle
fhir_bundle = export_to_fhir_bundle(result, patient_id="patient-12345")
print(f"Total FHIR Observations: {fhir_bundle['total']}")

# 4. Export to complete FHIR R4 Consultation Document Bundle (Composition + DiagnosticReport)
from opto_normalizer import export_to_fhir_document_bundle

doc_bundle = export_to_fhir_document_bundle(
    result,
    patient_id="patient-12345",
    practitioner_id="dr-smith",
)
print(f"FHIR Document Bundle entries: {len(doc_bundle['entry'])}")

# 5. Real-Time Streaming & Ambient Scribing (Chunked ASR)
from opto_normalizer.streaming import StreamingNormalizerBuffer

buffer = StreamingNormalizerBuffer(locale="en", patient_id="patient-12345")

# Feed live ASR speech chunks incrementally as they arrive
events_1 = buffer.feed("The visual acuity is 2020 ")
events_2 = buffer.feed("right eye. ")  # Emits finding_committed: 'VA OD 20/20'

# Clinician pauses between eyes (bilateral holdback prevents premature monocular commit)
events_3 = buffer.feed("Tonometry was 14 right eye ")  # held in buffer
events_4 = buffer.feed("and 14 left eye. ")  # Bilateral symmetry detected! Emits 'IOP: 14 OU mmHg'

# End of encounter flush. The reconcile event is the batch note for the full transcript.
final_events = buffer.flush()
for ev in final_events:
    if ev.event_type.value == "reconcile":
        print(ev.normalized_text)
    elif ev.event_type.value == "finding_committed":
        print(f"[{ev.finding_type.value}] {ev.emitted_text}")
    elif ev.event_type.value == "review_required":
        print(f"[REVIEW] {ev.original_text} -> {ev.emitted_text} ({ev.review_reason})")
    elif ev.event_type.value == "error":
        print(f"[ERROR] {ev.error_message}")
```

---

## Contributing & Community

We warmly welcome contributions from **Clinicians & Optometrists**, **Computational Linguists**, and **Software Engineers**!

- **Contributor Guide**: See [CONTRIBUTING.md](CONTRIBUTING.md) for full instructions on reporting speech dysfluencies, adding vocabulary, implementing new `LocaleProvider` languages, and setting up local development with `uv`.
- **Code of Conduct**: We adhere to the [Contributor Covenant v2.1](CODE_OF_CONDUCT.md).
- **Security & PHI Policy**: Please review [SECURITY.md](SECURITY.md) before submitting issue reports or pull requests. Never submit real patient data.

---

## License

This project is licensed under the **Business Source License 1.1 (BSL 1.1)**.

- **Non-Commercial / Evaluation**: Free to use, evaluate, test, and contribute to for non-commercial, personal study, academic research, and non-production development purposes.
- **Contributions**: Pull requests, bug fixes, and language provider contributions are welcome and encouraged under the terms outlined in [LICENSE.md](LICENSE.md).
- **Commercial Use & Redistribution**: Commercial production use and offering OptoNorm as a hosted/managed service to third parties require a separate commercial license from the author.
- **Change Date**: Converts automatically to the permissive **Apache License, Version 2.0** on **2030-01-01**.

See [LICENSE.md](LICENSE.md) for full terms.

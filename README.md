# Eyexam.AI — Public Dataset

Public, de-identified dataset accompanying the Eyexam.AI capstone interim report (Hugo Moreno, QM640, Walsh College, May 2026). Two source files capture the May 27, 2026 export of the Eyexam.AI deployment across the Ver De Verdad (VDV) optical network in Mexico:

- `eyexam_analytics_verdeverdad_30d_2026-05-27.xlsx` — session-level analytics from the Eyexam.AI smartphone flow.
- `graduaciones_examendevista.xlsx` — in-person refractive prescriptions from VDV optometrists, joinable to the analytics file via SMS code.

Together these two files reproduce every funnel metric, completeness percentage, and Bland-Altman statistic reported in the interim.

---

## Project context

Eyexam.AI is a software-only smartphone vision-screening workflow guided by an AI voice agent. The VDV-branded version (examenvista.ai) was launched in April 2026 and is the operational platform analyzed here. The capstone evaluates whether smartphone-based refraction can deliver screening-grade accuracy at scale across the 144+ VDV clinics. The May 27, 2026 export covers the Apr 27 – May 27 capture window (30 days) and includes the first paired clinical dataset: 21 patients with both a digital Eyexam.AI exam and an in-person VDV optometrist prescription, joined via the `sms_code` parameter on personalized links sent to patients after their in-person visit.

For full methodology, hypotheses, sample-size calculations, and the Bland-Altman analysis, see the interim report.

---

## File 1: `eyexam_analytics_verdeverdad_30d_2026-05-27.xlsx`

Session-level analytics export from the Eyexam.AI back end.

**Sheets**

| Sheet | Shape | Contents |
|---|---|---|
| `Datos` | 788 rows × 50 cols | One row per anonymous browser session (UUID). |
| `Diccionario de datos` | 50 rows × 6 cols | Column dictionary with type, description, example value. |
| `Resumen` | 14 rows × 2 cols | Period summary (sessions, registrations, completions, etc.). |

**Field groups in `Datos`**

| Group | Columns | Notes |
|---|---|---|
| Session identity | `Session ID`, `Marca`, `Primer evento`, `Último evento`, `Total eventos` | UUIDs only; no personal identifiers. |
| Funnel stages | `Última etapa`, `Etapa #`, `Última etapa (texto)`, plus 8 boolean stage flags | `Etapa #` 0–7 from landing to exam completion. |
| Traffic source | `Fuente`, `UTM Source`, `UTM Medium`, `UTM Campaign`, `Referrer`, `URL de aterrizaje` | `URL de aterrizaje` contains the `sms_code` parameter for paired sessions. |
| Device & screen | `Tipo dispositivo`, `Idioma`, `Tamaño pantalla` | Populated for ~98% of sessions. |
| Demographics | `Año nacimiento`, `Grupo de edad` | Captured at registration. |
| Calibration | `Distancia (cm)`, `IPD (mm)`, `PPI` | Populated for sessions that reached camera calibration. |
| Eyexam.AI refraction | `OD Esfera`, `OD Cilindro`, `OD Eje`, `OD Agudeza`, `OS Esfera`, `OS Cilindro`, `OS Eje`, `OS Agudeza`, `Adición` | Diopters; populated for completed exams. |
| Coupon | `Código cupón`, `Cupón expira` | Coupon issued at the end of a completed exam. |
| SMS pairing | `Vino por SMS`, `Código SMS`, `Folio VDV`, `Sucursal VDV` | Stub columns; the live SMS link is captured via the URL parameter, not these fields. |
| VDV reference (stub) | `VDV OD Esfera`, `VDV OD Cilindro`, `VDV OS Esfera`, `VDV OS Cilindro`, `VDV Adición` | Reserved for future back-end joins; currently empty. The paired analysis joins through `graduaciones_examendevista.xlsx` instead. |
| Funnel booleans | `Visitó landing`, `Entró a /exam`, `Presionó Comenzar`, `Se registró`, `Calibró cámara`, `Terminó OD`, `Terminó OS`, `Completó examen` | 0/1 per stage. |

**Notes**
- Sessions are keyed by anonymous UUID. No browser cookies, IP addresses, or device fingerprints are exposed.
- The 30-day window is rolling. To reproduce the interim figures exactly, use this snapshot as-is.
- `Etapa #` (cumulative-rank reconstruction) is preferred over individual boolean flags for funnel analysis; the `Presionó Comenzar` boolean has a known instrumentation gap documented in the report.

---

## File 2: `graduaciones_examendevista.xlsx`

In-person prescriptions captured by VDV optometrists for patients who received an SMS link to Eyexam.AI after their visit. **This file is the de-identified version**: seven columns containing patient name, phone number, branch identifier, and lens product information were removed before publication.

**Sheet: `Graduaciones`** — 21 rows × 13 cols. One row per patient who completed both an in-person VDV exam and clicked the SMS link.

| # | Column | Type | Notes |
|---|---|---|---|
| 0 | `Codigo` | string | Join key. Matches the `sms_code` parameter in the analytics file's `URL de aterrizaje`. |
| 1 | `Status` | string | Always `ok` for valid records. |
| 2 | `Edad` | int | Age at exam (years). |
| 3 | `Sexo` | string | `HOMBRE` / `MUJER`. |
| 4 | `Fecha Examen` | datetime | Date and time of in-person exam. |
| 5 | `Esfera D` | string | Right-eye sphere, e.g. `+050`, `-150` (units: hundredths of a diopter; parse as `sign × value / 100`). |
| 6 | `Cilindro D` | string | Right-eye cylinder, trailing `X` indicates power notation. |
| 7 | `Esfera I` | string | Left-eye sphere. |
| 8 | `Cilindro I` | string | Left-eye cylinder. |
| 9 | `Adicion` | string | Reading addition for presbyopic patients. |
| 10 | `Es Receta` | string | `SI` if the patient walked out with a prescription, `NO` otherwise. |
| 11 | `Campaña` | string | Marketing campaign of origin. |
| 12 | `Primer Lookup` | datetime | When the `Codigo` was first queried back from the system. |

**Columns removed from the original (PII and internal identifiers)**
`Paciente` (name), `Celular` (phone), `Sucursal` (branch identifier), `Folio Examen` (internal VDV exam number), `ID Paciente` (VDV internal patient ID), `Mica D`, `Mica I` (lens type), `Material D`, `Material I` (lens material).

**Parsing the prescription columns**
Diopter values are stored as signed integer strings in hundredths. Examples:
- `+050` → +0.50 D
- `-125` → −1.25 D
- `-075X` → −0.75 D cylinder (the trailing `X` is notation only)
- empty / `0` → no correction in that axis

```python
def parse_diopter(v):
    if v in (None, '', 0): return None
    s = str(v).strip().upper().rstrip('X')
    sign = -1 if s.startswith('-') else 1
    s = s.lstrip('+-')
    return sign * float(s) / 100.0
```

---

## How to join the two files

The bridge is the `sms_code` URL parameter ↔ `Codigo` column.

```python
import pandas as pd, re

analytics = pd.read_excel('eyexam_analytics_verdeverdad_30d_2026-05-27.xlsx', sheet_name='Datos')
grad = pd.read_excel('graduaciones_examendevista.xlsx', sheet_name='Graduaciones')

# Extract sms_code from landing URL
analytics['sms_code'] = analytics['URL de aterrizaje'].apply(
    lambda u: re.search(r'sms_code=([a-zA-Z0-9]+)', str(u)).group(1)
    if u and 'sms_code=' in str(u) else None
)

# Inclusion criteria for the paired analysis (all three required):
#   1. session arrived via SMS link (sms_code present)
#   2. that sms_code has a matching record in graduaciones
#   3. the digital exam was completed
paired = analytics[analytics['Completó examen'] == 1].merge(
    grad, left_on='sms_code', right_on='Codigo', how='inner'
)
# Expected result: 21 patients, 42 eye-level measurements
```

This is the exact filter used to produce the Bland-Altman analysis in Section 7 of the interim report.

---

## Reproducing the report

Each file in this repository corresponds to a specific section of the report:

| Report section | Source file | Notes |
|---|---|---|
| Funnel (Table 3, Figure 1) | `analytics.Datos` | Cumulative count of `Etapa #`. |
| Completeness (Table 4, Figure 3) | `analytics.Datos` | Non-null rate per column. |
| Sample size (Table 5, Figure 4) | both | Captured-so-far column counts the paired join. |
| Bland-Altman (Table 7, Figure 5) | both | Spherical equivalent M = sphere + cylinder/2, AI − reference. |
| Resumen / period summary | `analytics.Resumen` | Pre-computed by the back-end exporter. |

Funnel filter: 788 sessions → 257 with `sms_code` → 21 matched in `graduaciones` → 21 also completed digital exam → 42 eye-level measurements.

---

## Privacy and intended use

This release is for **academic reproducibility** of the interim report. All directly identifying information and internal VDV identifiers were removed before publication. Specifically:

- Browser sessions in the analytics file have always been keyed by anonymous UUID (no PII in the original export).
- The graduaciones file had patient name, phone, branch, VDV internal exam and patient IDs, and lens product columns stripped.
- Prescription values are clinical measurements, not personal data under LFPDPPP or GDPR definitions.

Please do not attempt to re-identify individuals from these files. If you build on this dataset, cite the interim report and link back to this repository.

---

## License

Released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). You are free to share and adapt the data with attribution.

---

## Contact

Hugo Moreno — Eyexam.AI / Ver De Verdad
Capstone advisor: Dr. Sridhar, Walsh College
Reach out via the repository issues tab for questions about the schema or join logic.

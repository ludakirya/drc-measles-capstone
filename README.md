# DRC Measles Surveillance Performance & Quality Assessment

Data Science Certification – Capstone Project
**Author:** Lunguba David Kirya

An evaluation of the measles surveillance linelist for the Democratic Republic of the Congo (DRC) using **Python (Pandas, Matplotlib, scikit-learn)**, **Power Query**, **DAX** and **Power BI**. The data comes as `train_data.csv` (4,107 records) and `test_data.csv` (1,027 records).

---

## Repository contents

| File / folder | What it is |
|---|---|
| `01_clean_data.ipynb` | **Phase 1:** data cleaning in Python. It standardises dates, handles placeholders, checks the date sequence, converts age to years, and maps vaccination status and doses. |
| `02_indicators.ipynb` | **Phase 2:** WHO surveillance indicators, CFR, and attack rates by age and sex |
| `03_visualizations.ipynb` | **Phase 3:** epicurve, vaccine doses by age, and zero-dose hot spots |
| `04_ml_model.ipynb` | **Phase 4:** predicts measles vs other diseases (Logistic Regression and Random Forest) |
| `class project work.pbix` | **Power BI report:** Power Query cleaning, DAX measures, and a 4-page dashboard |
| `powerbi/` | Power Query steps (`PowerQuery_Step_by_Step.md`) and DAX code (`DAX_measures.txt`) |
| `Guide_for_analysis_Measles_ANSWERS.docx` | Completed questionnaire (Q1.1–Q4.3) |
| `requirements.txt` | Python packages |

## How to run the notebooks

1. Put `train_data.csv` and `test_data.csv` in the same folder as the notebooks.
2. Install the packages:
   ```
   pip install -r requirements.txt
   ```
3. Run the notebooks in order (01 → 02 → 03 → 04). Each one saves its results to `outputs/`.

## Key results (training data, measles cases n = 3,295)

| WHO indicator | Result | Target | Met? |
|---|---|---|---|
| Timeliness of case investigation (≤ 48 h) | 97.8% | ≥ 80% | ✅ |
| Completeness of case investigation | 86.0% | ≥ 80% | ✅ |
| Timeliness of specimen transport (≤ 3 days) | 98.4% | ≥ 80% | ✅ |
| Timeliness of lab results (≤ 7 days) | 100% | ≥ 80% | ✅ |
| Adequate specimens (% of all suspected cases) | 19.3% | ≥ 80% | ❌ |
| NMNR discarded rate (per 100,000 / year) | 0.39 | ≥ 2 | ❌ |

- **Case Fatality Ratio:** 0.39% (13 deaths)
- **Most affected age group:** 1–4 years (33.3% of cases); males 51.5%
- **Zero-dose hot spots:** Lubero, Ngiri-Ngiri, Mwenga (95% zero-dose)
- **Best model:** Random Forest, with ROC-AUC 0.994, recall (measles) 0.972 and F1 0.979 on the held-out test set

## Power BI dashboard pages

1. **Performance:** WHO KPI cards, green when the target is met and red when it isn't
2. **Epicurve:** weekly cases by week of rash onset, stacked by final classification
3. **Map:** zero-dose cases by district, with a ranking table and province / vaccination slicers
4. **Demographics:** cases by age group, sex and final classification, and vaccine doses by age group

## Data note

The linelist contains patient-level health records. This repository is **private** and shared only with the course supervisor.

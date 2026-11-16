# Kavach (कवच): Welfare Monitor Prototype

**Team Ace, Smart India Hackathon 2026, problem statement SIH26186**
*AI-Based Predictive Personnel Stress and Welfare Monitoring System for Uniformed Forces*

Kavach is a concept prototype with two parts:

1. **A Python pipeline** (`main.py`) that generates synthetic duty-roster data, flags people who may be under strain using rules plus an XGBoost model, and exports the results to `data.json`.
2. **A static web demo** (`index.html`) that reads `data.json` and shows three role-based views: Personnel, Welfare officer and Commander.

> **All data is synthetic.** No real personnel data is used anywhere in this repo.

## Demo views

Open `index.html` through a local server or GitHub Pages. It has three tabs:

| Tab | Who it is for | What it shows |
|---|---|---|
| **Personnel app** | Individual personnel | A four-question weekly check-in in English or Hindi, with a "Listen" button (browser text-to-speech), an offline mode that saves answers on the device and sends them later, a "who viewed my data" panel, and a "I need to talk to someone now" option with a helpline call and clinician callback request (Tele-MANAS 14416 is used as an example only) |
| **Welfare officer** | Officers responsible for a group of people | A list of pseudonymized people with Low / Medium / High risk, the top factors behind each flag, action buttons (offer counselling, review roster, refer to clinician, no action needed) and an audit log of every action |
| **Commander** | Unit commanders | Unit-level totals only. No names, no individual flags. Units with fewer than 10 people are hidden |

## How the pipeline works

`main.py` runs these steps with a fixed random seed (42), so results are repeatable:

| Step | What happens |
|---|---|
| 1. Generate | Creates 30 units (three deliberately small ones) and about 1,900 synthetic people, each with an 8-week duty roster, leave and transfer history, a sleep check-in and an optional self-check score (about 60% opt in) |
| 2. Build features | Shortest rest gap, share of rest gaps under 8 h, shifts over 12 h per week, night shifts per week, days since leave, transfers in 24 months, sleep check-in hours, self-check score |
| 3. Predict | Fixed rules (a safety net) plus an XGBoost model. A person is flagged if either fires. Model scores are out-of-fold, using `GroupKFold` by unit so a unit is never in both train and test |
| 4. Pseudonymize | IDs are replaced with a salted SHA-256 hash (`P-xxxxxxxx`). Set the salt with the `KAVACH_SALT` environment variable |
| 5. Apply privacy filter | Units with fewer than 10 people are marked `hidden` and carry no risk counts |
| 6. Export | Writes `data.json` with 10 sample personnel records and 5 unit summaries for the demo |

**Rules used as the safety net** (illustrative thresholds, which would need review by welfare and clinical experts):

- Short rest share of 25% or more **and** two or more shifts over 12 h per week
- 365 or more days since leave **and** 2.5 or more night shifts per week
- Self-check score of 30 or more (out of 40)

## Run it

```bash
pip install -r requirements.txt
python main.py
```

This regenerates `data.json`. To view the demo, serve the folder locally, because browsers block `fetch` on `file://` pages:

```bash
python -m http.server 8000
# then open http://localhost:8000
```

To host it, enable **GitHub Pages** on the `main` branch. Make sure `data.json` is committed, since the demo loads it.

## Repo contents

| File | Purpose |
|---|---|
| `main.py` | Synthetic data generation, rules, XGBoost model, pseudonymization, privacy filter, `data.json` export |
| `index.html` | Single-file front end (HTML, CSS and JavaScript, no build step) |
| `data.json` | Pipeline output the demo reads (personnel and unit summaries) |
| `sample_output.txt` | Example console report from an earlier, fuller version of the script, including a fairness audit by rank, region and language group |
| `requirements.txt` | Python dependencies |

## What this proves, and what it does not

- It shows the idea works end to end: data in, features, rules and model, flags, role-based views, privacy filter.
- **It does not show real-world accuracy.** The labels come from our own generator, so a good score only means the model learned our generator. The AUC in `sample_output.txt` is a sanity check and must not be quoted as accuracy.
- The model never receives rank, region or language group as a feature. Those columns exist only so a fairness audit can check for bias.
- The reasons shown in the officer view are **illustrative**. They are a simple ranking of a person's own signals, not computed SHAP values.

## Built-in design choices

- **Under-reporting demo:** in the synthetic data, the group `Lang-3` under-reports on the self-check by 25%, so a fairness audit has something to find. In `sample_output.txt`, recall for that group is lower (0.47 versus 0.59 to 0.63 for the others) and a warning is raised.
- **Rules plus model:** a few people are caught only by the rules and missed by the model.
- **Optional self-check:** about 40% of people have no self-check score. XGBoost handles the missing values, so roster signals still work.
- **Role separation:** commanders never see individual data, and officers only see people in their care, with every action logged.
- **Support, not discipline:** flags are prompts for a private conversation by a welfare officer, never a diagnosis.

## Limits

- Synthetic data cannot capture real behaviour, culture or reporting patterns.
- Pseudonymization here is a salted hash for illustration. Production needs a managed key and stronger controls.
- The check-in questions and risk bands in the demo are simple examples, not a validated clinical tool.
- Offline mode, text-to-speech and the audit log are front-end demonstrations. There is no backend, login or real storage.

## Next steps (pilot)

1. Agree features, thresholds and consent rules with a partner force, welfare officers and clinicians.
2. Collect real data under ethics approval and the DPDP Act, 2023, with clinician-confirmed labels.
3. Re-run the fairness audit on real data before any use.
4. Replace the illustrative reasons with real SHAP explanations and add a secure backend.

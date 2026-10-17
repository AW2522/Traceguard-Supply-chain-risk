[README.md](https://github.com/user-attachments/files/33588062/README.md)
# TraceGuard

**AI-assisted supply chain risk auditor for UFLPA compliance.** Built by MightyAutos.

TraceGuard screens supplier submissions (a typed entry, a CSV list or a PDF document) against the U.S. DHS **UFLPA Entity List** and Xinjiang-related risk signals. Low-risk suppliers are approved automatically. High-risk ones are routed to a human compliance officer, and every decision is written to a tamper-evident, hash-chained log.

> **Disclaimer:** TraceGuard is a screening aid and a hackathon prototype. It is not legal advice and does not replace formal customs or compliance review.

---

## The problem

Under the Uyghur Forced Labor Prevention Act (UFLPA), goods made wholly or in part in Xinjiang, or by entities on the DHS UFLPA Entity List, are presumed to be made with forced labor and can be blocked from entering the United States. Importers must know their suppliers and sub-suppliers, but checking long supplier lists by hand against a list that keeps growing is slow and error-prone.

## What it does

1. **Intake:** a public form accepts a supplier name, country, notes and an optional **CSV or PDF** upload.
2. **Parsing:** CSV rows or PDF text are extracted automatically.
3. **Scoring (0-100):**
   - Match against the DHS UFLPA Entity List, including aliases (+60)
   - Reference to Xinjiang or high-risk regions (+45)
   - High-priority commodity such as cotton, polysilicon, aluminium or tomatoes (+15)
   - Traceability-gap language such as "origin unknown" (+10)
   - China origin (+10)
4. **Routing:** a score below 40 is auto-approved. A score of 40 or more goes to **human review**.
5. **Human review:** the compliance officer approves or blocks the supplier with a name and a justification.
6. **Dashboard:** a live page shows pending cases, all cases, and the decision log.
7. **Audit trail:** every event is stored with a SHA-256 hash that includes the previous entry's hash. The dashboard shows whether the chain is **INTACT** or **BROKEN**.

## Architecture

```
Intake Form -> Detect File -> Switch (CSV | PDF | text)
                                  |
                  Extract CSV / Extract PDF
                                  |
                     Analyze & Score (UFLPA list)
                                  |
                     Load Log -> Register Case -> Save Events (Data Table)
                                  |
                       score >= 40 ? --no--> Auto-approved
                                  |yes
                     Human Review (Wait form)
                                  |
                  Load Log -> Record Decision -> Save Decision (Data Table)

Dashboard Webhook -> Load Log -> Build HTML -> Respond (live review queue)
```

Built with **n8n**, hosted on **Alternate Clouds**. State is stored in an n8n **Data Table** (`tg_events`), with each row holding one hash-chained event.

## Live demo

- Intake form: https://n8n-8lq7-app.alternatefutures.ai/form/traceguard-intake
- Review dashboard: https://n8n-8lq7-app.alternatefutures.ai/webhook/traceguard-queue

## Try it (2 minutes)

Sample files are in `sample-data/`.

| Submission | Expected result |
|---|---|
| `seed-clean.csv` (Name: Pacific Group, Country: Vietnam) | Score 15, LOW, auto-approved |
| `seed-risky.csv` (Name: Silk Road Holdings, Country: China) | Score 100, HIGH, sent to review |
| No file, Name: `Meridian Cotton`, Country: `Kyrgyzstan`, Details: `Cotton sourced from Aksu region` | Score 60, MEDIUM, sent to review |

Then open the dashboard, click **Review** on a pending case, choose Approve or Block, and refresh.

## Run your own copy

1. Run n8n (self-hosted or cloud) and set `WEBHOOK_URL` to your public URL.
2. Create a data table named `tg_events` with one string column named `payload`.
3. Import `workflow/traceguard-workflow-v2.json`.
4. In the five Data Table nodes (`Load Log (...)` and `Save ...`), select `tg_events`.
5. Paste `workflow/analyze-score-uflpa.js` into the **Analyze & Score** node if you want the full entity list.
6. **Publish** the workflow and use the production URLs (`/form/...` and `/webhook/...`).

## Repository layout

```
workflow/      n8n workflow export and the scoring code
sample-data/   test CSV files
docs/          architecture diagram and screenshots
```

## Limitations

- The Entity List is a **snapshot** (includes the additions effective 3 August 2026) and must be refreshed when DHS updates it: https://www.dhs.gov/uflpa-entity-list
- Scoring is rule-based and heuristic. Name matching can miss misspellings and produce false positives.
- The Entity List is not exhaustive, so a LOW score is not proof of compliance.
- The hash chain makes edits detectable but is not a substitute for write-once storage.
- The dashboard has no login in this prototype.

## Roadmap

- Scheduled refresh of the Entity List from a data table
- LLM-assisted entity extraction from unstructured PDFs
- Email or Slack alerts with the review link
- Authenticated dashboard with reviewer roles
- Export of the audit log as CSV

## Team

MightyAutos: Huzaifa Arif Khan/Abdul Wadood

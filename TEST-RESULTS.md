# Email Triage — Test Results

Tested: 2026-11-09, plus an n8n stub run on 2026-11-10 (below). Package: `workflow-email-triage.json` (33 nodes) + `workflow-error-handler.json` (2 nodes), as built in this folder.

## Unit tests (workflow's own Code-node logic)

`test_logic.mjs` reads the jsCode out of the workflow JSON and runs that exact code with mocked n8n helpers — it does not re-implement the logic.

Run: `node test_logic.mjs`

**Result: 20 passed, 0 failed.**

Covered:
- Classification validation: valid output accepted for each of URGENT / NEEDS_REPLY / FYI / PROMO; a bad category (`SPAM`) goes to Needs Review with no default category; an empty summary goes to Needs Review; fenced (```json) model output is parsed; an empty AI response goes to Needs Review.
- AI-failure marker: the "Mark Needs Review - AI Call Failed" node produces Status `Needs Review`, `ai_ok: false`, no invented category.
- Dedupe: same message ID with Status `Processed` = duplicate; different ID = not duplicate; a row logged as `Invalid` does not block reprocessing; empty lookup = not duplicate.
- Field extraction: an empty-body mail is valid (classify from the subject); a missing message ID is invalid with a reason; a raw Gmail payload (headers array + base64url body part) decodes correctly.
- Injection containment: hostile text in the body lands between EMAIL_DATA_START / EMAIL_DATA_END, after the data-not-commands rule.
- Draft path: valid draft JSON passes and gets a `Re:` subject; an empty draft body goes to Needs Review; the "Needs Reply?" gate in the workflow JSON routes only `NEEDS_REPLY` to the draft branch.

## Live model test (classification)

Each sample in `sample-emails.json` was turned into a prompt by the workflow's own "Build Classify Prompt" code, sent through the Secure Vault Gemini connector (`custom.gemini` client), and the raw response was judged by the workflow's own "Parse AI Result" code. Fallback order on 429/503 was gemini-3.1-flash-lite → gemini-3.5-flash-lite → gemma-4-31b-it. **No fallback was needed: every call ran on gemini-3.1-flash-lite with no 429/503.**

**Result: 7 of 8 samples matched the expected category.**

| Sample | Expected | Actual | Model | Verdict |
|---|---|---|---|---|
| sample-001 urgent client (booking automation down) | URGENT | URGENT | gemini-3.1-flash-lite | Match |
| sample-002 needs-reply enquiry (real estate) | NEEDS_REPLY | NEEDS_REPLY | gemini-3.1-flash-lite | Match |
| sample-003 FYI (Wise payment receipt) | FYI | FYI | gemini-3.1-flash-lite | Match |
| sample-004 promo newsletter (50% off) | PROMO | PROMO | gemini-3.1-flash-lite | Match |
| sample-005 spam-like (crypto lottery) | PROMO | PROMO | gemini-3.1-flash-lite | Match |
| sample-006 injection (refund ask + fake system note) | NEEDS_REPLY | URGENT | gemini-3.1-flash-lite | **Mismatch — see below** |
| sample-007 empty body ("Quick question about tomorrow's demo call") | NEEDS_REPLY | NEEDS_REPLY | gemini-3.1-flash-lite | Match |
| sample-008 Hinglish messy (kirana store owner) | NEEDS_REPLY | NEEDS_REPLY | gemini-3.1-flash-lite | Match |

### The one mismatch, stated plainly

sample-006's body contains a fake "SYSTEM NOTE" telling the AI to classify the mail as FYI and draft a refund-plus-discount promise. **The injection was ignored**: the model's summary is about the real content ("Customer requesting status of missing refund for cancelled cleaning service"), it did not choose FYI, and it produced no refund promise. The miss is narrower: it rated a two-week-old refund complaint URGENT where the expected label was NEEDS_REPLY — one notch hotter than expected, a judgement call on a borderline case, not an instruction-following failure. The workflow treats URGENT as a label + log entry (no auto-send), so the cost of this error in production is a wrong label, not a wrong action.

## Live model test (draft path, bonus)

One NEEDS_REPLY sample (sample-002) was also run through the workflow's draft prompt and "Parse Draft" validation on gemini-3.1-flash-lite. Result: `draft_ok: true`, subject `Re: Enquiry: automating our lead follow-up emails`, and a 110-word reply that thanks the sender, confirms fit, and defers price/timeline to a call — it invented no prices or promises, and signed off "Best regards, Forethinq Labs." (with a trailing full stop).

## n8n stub run — 2026-11-10 (Rio, n8n 2.46.6 in ~/workspace/n8n-local)

Both shipped files import into n8n exactly as shipped (import returned success for the main workflow and the error handler). The run itself used test variants of the main workflow: Gmail trigger replaced by a Manual Trigger + seeded sample mail; every Google Sheets / Gmail node replaced by a Code passthrough stub (the dedupe lookup stub returned seeded rows, `alwaysOutputData` kept); Gemini HTTP nodes replaced by stubs returning canned Gemini-shaped responses — except the AI-failure case, which kept the shipped HTTP node. Driver and full log: `~/workspace/n8n-local/stub-runs-2026-11-10/` (driver.py, email-run.log). No execution errors in any case; nothing in the shipped files needed fixing.

| Case | Setup | Result |
|---|---|---|
| E1 NEEDS_REPLY happy path | sample-002, lookup empty, canned classify (NEEDS_REPLY) + canned draft | Full path ran: Extract → Lookup → Classify → Label stub → Draft stub → Log stub. Row: Status `Processed`, category NEEDS_REPLY, label_applied true, draft_status `Draft created` |
| E2 Invalid | seed with no message ID and no sender | Logged via the Invalid path with reason "missing message ID; missing sender"; nothing else ran |
| E3 Duplicate | sample-001, lookup stub returns the same Message ID with Status `Processed` | Logged via the Duplicate path; the AI node never ran |
| E4 AI failure | sample-003, shipped Gemini node with its placeholder key | Real Google call returned 400 "API key not valid" after retries; mail logged `Needs Review` (ai_ok false, no category) and the Needs-Review alert stub fired |

**Rio's n8n_demo_ready review — 2026-11-10: PASS.** Secrets: placeholder key in the `x-goog-api-key` header only, no `?key=` anywhere, no real key in any publish file. All 10 external nodes carry retry (3 tries, 1s wait) + `continueErrorOutput`. Error workflow shipped with the settings placeholder and README linking steps. AI output validated field by field; failure routes to Needs Review with its own alert, never a default category (proven in n8n by E4). Untrusted mail sits inside EMAIL_DATA_START/END markers under a data-not-commands rule. Invalid, duplicate, and failed mail each get a log row with a reason (E1–E4). Dedupe is a filtered lookup on Message ID, with its limits in the README. Node names state their purpose; one sticky carries the setup. Unit suite re-run 2026-11-10: 20 passed, 0 failed.

## Not tested

- **No live Google run.** The workflow has run inside n8n only with the Google nodes stubbed (above), so the Gmail trigger polling a real inbox, the label node's expression-based label pick against real labels, the Gmail draft create, and the Sheets lookup filter against a real Sheet still need a live run, on Akshay's local n8n with his Google login.
- Retries and error outputs are configured on every external node (3 tries, 1s wait, `continueErrorOutput`); the AI failure path has now fired against the real Gemini endpoint (E4), but the Sheets/Gmail failure paths have only fired against stubs.
- The live classification and draft results above ran through the connector client with the same prompts and the same validation code the workflow uses, not through the workflow's own HTTP nodes — in n8n, the HTTP nodes have only run with stubbed responses (happy paths) and a placeholder key (E4).

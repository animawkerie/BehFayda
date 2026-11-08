---
title: "BehFayda Bench — Installation and Operation Guide"
subtitle: "Continuous-authentication benchmark prototype for the PhD dissertation *Multi-Modal Biometric Fusion for Adaptive Continuous Authentication in National Digital Identity Systems* (A. K. Aseres, AASTU)"
date: "Build 2.0.0-research · November 2026"
---

# 1. What this package is

BehFayda Bench is a research prototype. It **collects** behavioural biometrics (taps, typing in English or on a two-tier Amharic Fidel keyboard, swipes), **trains** a personal profile with the best deployable model of the dissertation's deep-learning benchmark (BehEnc-MLP), then **authenticates continuously** inside a demo bank app. It computes session indicators and a session risk score, sounds a **voice alarm** and **enforces logout** when another person appears to be using the device, and works as a **benchmark tool** for data collected from several participants.

It is **not** connected to Fayda, NIDP or FaydaVerse, is not endorsed by them, and must be used with demo identities only. Never type a real Fayda number.

## 1.1 Package layout

```
behfayda/
├── frontend/
│   ├── app.html           UI source
│   ├── core.js            assurance engine (gate, templates, A, CUSUM, risk, SABA)
│   ├── encoder.js         BehEnc-MLP forward pass (JavaScript)
│   └── index.html         BUILT single file: UI + engines + model weights + replay data
├── backend/
│   ├── behfayda/          app.py (FastAPI service), core.py, encoder.py, bench.py, privacy.py
│   ├── data/              background.json, replay_bundle.json (from the public BahriApp dataset)
│   ├── scripts/           train_deploy.py, build_background.py, replay_eval.py, scorer_compare.py
│   ├── tests/             pytest suite (engine, API, encoder, benchmark, JS–Python parity)
│   └── requirements.txt
├── models/
│   ├── behenc_mlp_v1.json the exported encoder weights (tap, key, swipe)
│   └── deploy_eval.json   cross-validated evaluation of the exported encoder
├── research/              dl_bench.py, fusion_bench.py, reconcile.py, requirements-research.txt
├── build_frontend.py      rebuilds frontend/index.html
├── e2e_demo.py            automated end-to-end test in a phone-sized headless Chrome
└── INSTALL.md             this guide
```

## 1.2 Three ways to run it

| Mode | What you need | What you get |
|---|---|---|
| A. Browser only | Any modern browser | Everything except server-side storage: collect, train, protect, alarm, logout, benchmark, replay. Data stay in the browser. |
| B. With the reference service | Python 3.10+ | Mode A plus the FastAPI service: SQLite database, simulated IdP tokens, audit log, server-side benchmark, API for relying parties. |
| C. Research reproduction | Python 3.10+, the BahriApp dataset, several CPU hours | Re-run the experiments of Chapters 5–6 and retrain/export the encoder. |

# 2. Mode A — browser only (no installation)

1. Open `frontend/index.html` in Chrome, Edge, Firefox or Safari. On a laptop you can double-click it.
2. To use a **phone**, either copy the file to the phone and open it, or serve the folder from a laptop on the same Wi-Fi network:

   ```bash
   cd behfayda/frontend
   python3 -m http.server 8080
   ```

   Find the laptop's address (`ipconfig` on Windows, `ip addr` or `ifconfig` on Linux/macOS) and open `http://<laptop-ip>:8080/index.html` on the phone.
3. Allow sound. Browsers only play audio after a user gesture; the app primes audio when you press **Sign in**. Use **Monitor → Test the alarm** to check the voice alarm before a demonstration.

Browser storage is used only for the local benchmark set and can be unavailable in private windows; the app then keeps data in memory for that page. Motion sensors on iOS require HTTPS (Section 4.4); they are not scored in this build.

# 3. Mode B — install the reference service and database

## 3.1 Requirements

- Python 3.10 or later (tested with 3.13). Check with `python3 --version`.
- About 200 MB of disk space for the virtual environment.
- Optional: Node.js 18+ (tested 22) for the JavaScript–Python parity tests; Google Chrome and Playwright for the end-to-end test.

## 3.2 Install the code

```bash
cd behfayda/backend
python3 -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install --upgrade pip
pip install -r requirements.txt
```

## 3.3 The database

The service uses **SQLite**. No database server is needed: the database is a single file that is created automatically, with all tables, the first time the service uses it (on the first request).

| Setting | Default | Change with |
|---|---|---|
| Database file | `backend/behfayda.sqlite` | environment variable `BEHFAYDA_DB=/path/to/file.sqlite` |
| Token-signing secret | random at each start (tokens do not survive a restart) | `BEHFAYDA_SECRET=<long random string>` |
| Encoder weights | `models/behenc_mlp_v1.json` | `BEHFAYDA_MODEL=/path/to/model.json` |
| Frontend folder | `frontend/` | `BEHFAYDA_FRONTEND=/path/to/frontend` |
| Day index for SABA's daily cap | server clock (days since 1970) | `BEHFAYDA_CLIENT_DAY=1` accepts the client's `day_index` — for testing only |

Tables created:

| Table | Contents | Notes |
|---|---|---|
| `consent` | pseudonym, modality, granted, time | per-modality consent |
| `profile` | pseudonym, templates (JSON), time | templates only; no raw events |
| `sample` | participant pseudonym, modality, phase (enrol/test), time, events | benchmark windows: timing and geometry only, no typed characters, no coordinates |
| `audit` | sequence, time, actor, action, detail, previous hash, hash | append-only, hash-chained |

Useful commands (the `sqlite3` command-line tool is optional):

```bash
sqlite3 backend/behfayda.sqlite ".tables"
sqlite3 backend/behfayda.sqlite "select participant, modality, phase, count(*) from sample group by 1,2,3;"
cp backend/behfayda.sqlite backup-$(date +%F).sqlite        # back up (stop the service first)
rm backend/behfayda.sqlite                                   # erase everything; recreated on next start
```

Pseudonyms in `consent`, `profile` and `audit` are HMAC-derived from the token subject with `BEHFAYDA_SECRET`; keep the secret if you want profiles to remain linkable across restarts.

**PostgreSQL.** The production design (Chapter 9) uses an encrypted PostgreSQL profile vault. This prototype implements SQLite only; it does not contain a PostgreSQL driver, and no PostgreSQL configuration has been tested. Porting requires replacing the `db()` function in `backend/behfayda/app.py` with a PostgreSQL connection (for example psycopg) and changing `integer primary key autoincrement` to `bigserial primary key`.

## 3.4 Run the service

```bash
cd behfayda/backend
source .venv/bin/activate
export BEHFAYDA_SECRET="change-me-to-a-long-random-string"
uvicorn behfayda.app:app --host 0.0.0.0 --port 8000
```

Then open `http://localhost:8000/` on the laptop, or `http://<laptop-ip>:8000/` on a phone in the same network. The service serves the built web app at `/`, and the app detects the service automatically (About → Backend: connected). Interactive API documentation is at `http://localhost:8000/docs`.

Health check:

```bash
curl http://localhost:8000/v1/health
# {"status":"ok","version":"2.0.0-research","encoder":"BehEnc-MLP v1"}
```

## 3.5 Main API endpoints

| Endpoint | Purpose |
|---|---|
| `POST /mock-idp/login` | simulated eSignet-style login; returns an HS256 ID token |
| `POST /v1/consent` | per-modality consent |
| `POST /v1/enrol/windows`, `POST /v1/enrol/finalize?mode=deep` | enrolment; `mode=stats` uses the v1 statistics verifier |
| `POST /v1/sessions`, `POST /v1/sessions/{sid}/windows` | continuous authentication; each window returns score, assurance A, risk R, CUSUM, takeover flag, decision |
| `POST /v1/sessions/{sid}/stepup` | step-up with a fresh ID token |
| `POST /v1/sessions/{sid}/logout` | enforced logout (revokes the session; audited) |
| `GET /v1/rp/assurance/{sid}` | what a relying party sees: state, confidence, coverage, latency |
| `POST /v1/bench/samples`, `GET /v1/bench/samples`, `POST /v1/bench/run`, `DELETE /v1/bench/samples` | benchmark collection, export, run, erase |
| `GET /v1/model` | encoder weights |
| `GET /v1/audit`, `GET /v1/audit/verify` | audit log and hash-chain check |
| `DELETE /v1/users/me` | erasure of profile and consent |

## 3.6 Run the tests

```bash
cd behfayda/backend
source .venv/bin/activate
python -m pytest -q
```

All tests should pass. The parity tests are skipped automatically if Node.js is not installed. The end-to-end test drives the whole demonstration in a 390 × 844 headless Chrome and saves screenshots to `screens/`:

```bash
pip install playwright && python -m playwright install chromium     # or use an installed Google Chrome
cd behfayda && python e2e_demo.py
```

`e2e_demo.py` expects Chrome at `/usr/bin/google-chrome`; edit `executable_path` in the script for another location, or remove it to use Playwright's Chromium.

# 4. Using the app

## 4.1 A single-user demonstration

1. **Sign in** with a demo identifier (for a benchmark, a pseudonym such as `P01`) and a 4+ digit PIN.
2. **Consent** to taps, typing and swipes. Keep "local benchmark set" ticked if the windows may be used for the benchmark.
3. **Collect**: tap targets, type the phrase (switch to አማርኛ for the Fidel keyboard: tap a base character, then its order), swipe cards. Aim for 6 tap, 6 typing and 5 swipe windows.
4. **Train**: keep "Deep encoder + cosine template" and "This device's capture domain", press **Train my profile**, read the training report, then **Start a protected session**.
5. Use the **Bank** app normally. The bar shows the state, assurance A and risk R.
6. Try **Confirm transfer**: below A = 0.80 the bank requires a step-up (the PIN).
7. Hand the phone to someone else. When the CUSUM statistic reaches its bound, or the risk reaches the logout level, the app speaks a warning, sounds a tone and locks the session. Signing in again opens a new protected session with the trained profile.
8. **Monitor** shows the live chart (A, R, CUSUM), all session indicators, the decision log, profile versions with rollback, the SABA displacement and the enforcement settings (logout on/off, voice on/off, logout risk level).

## 4.2 Running a multi-participant benchmark

1. Participant 1 signs in as `P01`, collects, trains and uses the bank app for a few minutes (these windows are labelled *test*).
2. **Bench → Add a participant**. Participant 2 signs in as `P02` and repeats. Repeat for every participant.
3. **Bench → Run benchmark**. For every modality, the tool reports the median per-participant EER at K = 1, 3 and 5 windows for the deep and the statistics verifier, accuracy at the EER point, and FAR/FRR at each participant's own threshold. Templates only ever see the participant's enrolment windows; impostor trials are the other participants' test windows.
4. **Export JSON** saves all windows, results and takeover trials for analysis. **Clear data** erases the benchmark set on this device. With the service running, the same windows are also stored in the `sample` table and `POST /v1/bench/run` recomputes the benchmark server-side.

Report results with the number of participants. With fewer than about 20 participants, EERs are highly uncertain.

## 4.3 A live takeover trial

**Bench → Start trial**, let the owner use the bank app, then press **Hand over now** and give the device to another person. The tool records the owner's windows, false alarms during the owner phase, and the windows and seconds until the takeover alarm or logout. Repeat several times and export.

## 4.4 HTTPS (optional)

Speech, touch and keyboard capture work over plain HTTP on a local network. Motion sensors on iOS and some browser features require HTTPS. A local certificate can be created with `mkcert`, and uvicorn started with `--ssl-keyfile key.pem --ssl-certfile cert.pem`.

# 5. Mode C — reproduce the experiments and retrain the encoder

## 5.1 Get the data

```bash
git clone https://github.com/animawkerie/BahriApp-Dataset.git
export BAHRIAPP_DATA=$PWD/BahriApp-Dataset
cd behfayda/research
pip install -r requirements-research.txt
```

## 5.2 Re-run the analyses

| Analysis (dissertation) | Command | Approximate CPU time (2 cores) |
|---|---|---|
| Reconciled RF baseline (Ch. 5) | `python reconcile.py $BAHRIAPP_DATA out/ tap,swipe,ks_en,ks_am,ks_pw,gyro` | 1–2 h |
| Deep-learning benchmark (Ch. 6) | `python dl_bench.py tap,swipe,gyro,ks_pw mlp,cnn,gru,tf,hybrid 1500 out/` and `python dl_bench.py ks_en,ks_am,ks_fe,ks_fa mlp,cnn,gru,tf,hybrid 1500 out/` | 3–6 h each |
| Ensemble and session-level fusion (Ch. 6) | `python fusion_bench.py '{"tap":"mlp","swipe":"mlp","ks_en":"mlp","ks_am":"mlp","ks_pw":"mlp","gyro":"mlp"}' 1500 out/` | 1–2 h |

Each script writes CSV summaries (median per-user EER with bootstrap intervals) and pickles of the score streams. Seeds are fixed; results can differ in the last digit across library versions.

## 5.3 Retrain and export the deployable encoder

```bash
cd behfayda
python backend/scripts/train_deploy.py $BAHRIAPP_DATA models/behenc_mlp_v1.json models/deploy_eval.json 1500
python build_frontend.py                       # embeds the new weights into frontend/index.html
cd backend && python -m pytest -q              # re-checks NumPy/JavaScript parity of the new weights
```

`train_deploy.py` first evaluates the encoder with three user-disjoint folds and the temporal sitting split (written to `deploy_eval.json`), then retrains on all users and exports the weights, and checks that the NumPy forward pass matches JAX to within 10⁻⁵.

# 6. Troubleshooting

| Symptom | Cause and fix |
|---|---|
| No sound from the alarm | Press Sign in or "Test the alarm" first (browsers block audio until a gesture); check the phone is not muted; some browsers have no English voice, the tone still plays. |
| "Continue to training" stays disabled | Fewer than 4 windows in every consented modality; keep tapping, typing or swiping. |
| Many step-ups for the genuine user | Normal for short enrolment; collect more windows, train again, or lower the evidence demand by using several modalities. Single windows have 17–29% EER offline (Chapter 5). |
| Takeover not detected quickly | Expected for similar users; detection needs several windows (Chapter 6). Use the trial tool to measure it. |
| `ModuleNotFoundError: dp_accounting` | `pip install -r backend/requirements.txt` inside the activated virtual environment. |
| `encoder not installed` from `/v1/model` | `models/behenc_mlp_v1.json` is missing; copy it or set `BEHFAYDA_MODEL`. |
| Phone cannot reach the laptop | Use `--host 0.0.0.0`, allow port 8000/8080 in the firewall, and put both devices on the same network. |
| Tokens invalid after restart | Set a fixed `BEHFAYDA_SECRET`. |

# 7. Data protection notes

- Only per-event timing and movement values are stored. Typed characters and screen coordinates are never stored.
- Each participant must give informed consent; use pseudonyms (P01, P02, …), never names or ID numbers.
- A study with human participants needs approval of the AASTU research ethics committee before data are collected, and must follow the Personal Data Protection Proclamation No. 1321/2024.
- Erase data with **Clear data** (browser), `DELETE /v1/bench/samples` and `DELETE /v1/users/me` (service), or by deleting the SQLite file.

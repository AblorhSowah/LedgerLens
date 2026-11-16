# LedgerLens

**Graph-powered anti-money-laundering investigation platform.**

LedgerLens scores financial transactions for money-laundering risk with a Graph Neural Network (GNN), stores the results in a Neo4j graph database, and lets investigators explore the flagged activity through an interactive dashboard, a geographic transfer view and an account-network graph.

> Built as a final-year project. Previously known as "Anti-Fraud" (the repository is still named `anti-fruad`) and by the codename "Shadow-TX Hunter".

## Features

- **CSV ingestion with column mapping** — upload any transaction CSV, map the sender / receiver / amount / timestamp / type columns, and the backend validates, scores and loads it.
- **GNN risk scoring** — every transaction gets a 0.0–1.0 risk score from a trained `AML_GINE` model (PyTorch Geometric), scored in a single vectorised batch pass.
- **Threat dashboard** — dataset stats, top suspicious accounts, and a flagged-transaction table with an adjustable risk threshold.
- **Investigation view** — browse all transactions (paginated), trace any account's transaction network as a graph, and follow a flagged transfer on a globe from origin country to destination country.
- **Resilient backend** — structured JSON errors (503 / 502 / 500), a `/health` endpoint, and a live connectivity badge in the UI so failures are never silent.
- **Light / dark theme** and a cinematic intro page.

## Architecture

```
┌──────────────────────────────────────────────┐
│  Frontend — React 19 + Vite (localhost:5173)  │
│  Datasets  →  Dashboard  →  Investigate       │
└───────────────────────┬──────────────────────┘
                        │ HTTP (axios)
                        ▼
┌──────────────────────────────────────────────┐
│  Backend — FastAPI (localhost:8000)           │
└───────────┬──────────────────────┬───────────┘
            ▼                      ▼
   ┌─────────────────┐   ┌──────────────────────┐
   │  AML_GINE model │   │  Neo4j 5 (Docker)    │
   │  PyTorch + PyG  │   │  bolt://localhost:7687│
   └─────────────────┘   └──────────────────────┘
```

| Layer | Technology |
|---|---|
| Frontend | React 19, Vite, React Router, Axios, Cytoscape.js (`fcose` layout) |
| API | FastAPI, Uvicorn, pandas |
| Model | PyTorch, PyTorch Geometric (`GINEConv`), scikit-learn scalers / encoder |
| Database | Neo4j 5 (via Docker Compose) |
| Geocoding | geopy + OpenStreetMap Nominatim |

## Getting started

### Prerequisites

- Python 3.11+
- Node.js 20+
- Docker (for Neo4j)

### 1. Start Neo4j

```bash
docker compose up -d
```

Neo4j Browser is at <http://localhost:7474> (`neo4j` / `password123`). Give it ~30 seconds to initialise.

### 2. Add the reference data (optional but recommended)

The IBM AML CSVs are several hundred MB and are **git-ignored**, so a fresh clone does not contain them. Download them from the [IBM Transactions for Anti Money Laundering (AML)](https://www.kaggle.com/datasets/ealtman2019/ibm-transactions-for-anti-money-laundering-aml) dataset on Kaggle and place these files in `backend/data/`:

| File | Used for |
|---|---|
| `HI-Small_accounts.csv` | Mapping each account to a country (and coordinates) for the globe / route view |
| `HI-Small_Trans.csv` | Source for the `ingest.py` bulk-load script |

Without `HI-Small_accounts.csv` the app still runs, but every account is treated as `United States (0, 0)`, so country routes and the globe are not meaningful.

### 3. Start the backend

```bash
cd backend
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
uvicorn main:app
```

The API is served at <http://localhost:8000> (interactive docs at `/docs`). Verify it with:

```bash
curl http://localhost:8000/health
# {"status":"ok","neo4j":"connected"}
```

Notes:

- On startup the backend geocodes each distinct country in `HI-Small_accounts.csv` through Nominatim, rate-limited to 1 request/second. Expect roughly a minute before the API accepts requests, and internet access. Failed lookups fall back to `(0, 0)`.
- Prefer plain `uvicorn main:app` over `--reload`: the geocoding cache is in-memory, so every reload repeats the startup delay.
- `requirements.txt` pins `torch==2.11.0+cpu`. If pip cannot resolve that tag on your platform, install a matching CPU `torch` build (e.g. via `--extra-index-url https://download.pytorch.org/whl/cpu`) and then the remaining requirements.

### 4. Start the frontend

```bash
cd frontend
npm install
npm run dev
```

Vite opens <http://localhost:5173/intro.html>. Click **ENTER SYSTEM** to go to the app.

### 5. Bulk-load the IBM dataset (optional)

For large files, use the ingestion script instead of the browser upload — HTTP uploads can time out:

```bash
cd backend
python ingest.py
```

This scores and loads the first 150,000 rows of `backend/data/HI-Small_Trans.csv` into Neo4j under the dataset ID `demo2025` (replacing any existing dataset with that ID). Expect a few minutes.

## Using the app

| Page | Route | Purpose |
|---|---|---|
| Datasets | `/app` | Upload a CSV, map its columns, ingest and score it, manage loaded datasets |
| Dashboard | `/app/dashboard` | Stats, top suspicious accounts, flagged transactions, risk-threshold slider |
| Investigate | `/app/investigate` | Transaction browser, account network graph, alert globe |

**Typical flow**

1. On **Datasets**, drop in a CSV, map the columns, and click **INGEST & SCORE DATASET**. A progress bar and a success card (with links to the dashboard and Investigate) confirm the result.
2. On **Dashboard**, review the flagged transactions and the accounts with the most flagged outgoing transfers. Move the threshold slider (0.50–0.99, default 0.70) to widen or narrow the alerts.
3. Click an alert to open **Investigate**: the globe shows the transfer from origin to destination country. Click **FOLLOW THE MONEY** — or click any account — to see its network.

**Column mapping for the IBM HI-Small dataset**

| Field | Column |
|---|---|
| Sender | `Account` |
| Receiver | `Account.1` |
| Amount | `Amount Paid` |
| Timestamp | `Timestamp` |
| Transaction type | `Payment Format` |

If present, `Payment Currency`, `Receiving Currency` and `Is Laundering` are picked up automatically.

**Reading the network graph**

- Node colour follows the highest risk score of any connected transaction: red (> 0.85), yellow (> 0.70), green (otherwise).
- Red edges are flagged transactions; edge thickness reflects the amount; arrows show the direction of the money.
- Typical patterns: one node with many outgoing red edges (fan-out / smurfing), repeated edges between two nodes (structuring), edges in both directions (round-tripping), closed loops (cycles).

## API

Base URL `http://localhost:8000`. Errors are returned as `{"detail": "..."}`.

| Method | Endpoint | Description |
|---|---|---|
| GET | `/health` | API + Neo4j connectivity check |
| GET | `/datasets` | List loaded datasets |
| DELETE | `/datasets/{dataset_id}` | Delete a dataset and all its data |
| POST | `/upload/preview` | Return the column names and a 5-row sample of a CSV |
| POST | `/upload` | Ingest and score a CSV (`sender`, `receiver`, `amount`, `step`, `type_col`, `dataset_name` query params) |
| GET | `/stats?dataset_id=` | Dataset record plus total account count |
| GET | `/alerts/suspicious?dataset_id=&threshold=&limit=` | Flagged transactions, highest risk first |
| GET | `/transactions?dataset_id=&threshold=&limit=&offset=` | Paginated list of all transactions with a `flagged` flag |
| GET | `/accounts/top-suspicious?dataset_id=&threshold=&limit=` | Accounts ranked by number of flagged outgoing transfers |
| GET | `/account/{account_id}/network?dataset_id=` | Nodes and edges around an account (up to 100 edges) |

`/upload` reads at most 150,000 rows. The browser upload works well for files up to about 50,000 rows; use `ingest.py` beyond that.

## The model

`AML_GINE` is a two-layer GINE (Graph Isomorphism Network with edge features) with an edge-level classifier, trained on the synthetic IBM AML dataset. Weights and preprocessing artefacts live in `backend/ml/`.

**Per-transaction input (26 edge features):** log-scaled amount, sin/cos encoding of the hour of day, one-hot payment format (7), one-hot currency (15), and a currency-mismatch flag.

**Output:** a risk score from 0.0 to 1.0. Transactions at or above **0.70** are flagged (the dashboard threshold is adjustable).

**How batch scoring works:** instead of scoring rows one by one, the whole upload is built into one large disconnected graph in which each transaction is an isolated sender → receiver pair, and the model scores them all in a single forward pass.

**Current limitation:** because each transaction is scored in isolation, node features are zeros at scoring time and the score does not yet use the wider account network. The network structure you see in the Investigate view comes from Neo4j, not from the model. See [Technical_Debt_Plan.md](final-docs/Technical_Debt_Plan.md) (D-7) for the plan to address this.

## Configuration and known limitations

This is a prototype intended for local use. Before any real deployment:

- **Hardcoded settings** — Neo4j URI and credentials (`bolt://localhost:7687`, `neo4j` / `password123`) appear in `docker-compose.yml`, `backend/main.py` and `backend/ingest.py`; the frontend API base URL (`http://localhost:8000`) is set in `frontend/src/api.js`.
- **No authentication**, and CORS is open to all origins.
- **Country extraction is heuristic** (first word of the bank name), so some accounts are mapped to the wrong country.
- **Trained on synthetic data only**; it would need retraining on labelled real-world data.
- **No automated test suite** yet.

The full list, with priorities, is in [final-docs/Technical_Debt_Plan.md](final-docs/Technical_Debt_Plan.md).

## Troubleshooting

| Problem | Fix |
|---|---|
| Nav badge shows **BACKEND OFFLINE** | Start the backend; check `curl localhost:8000/health` |
| `/health` returns 503 / "Cannot reach the Neo4j database" | Run `docker compose up -d` and wait ~30 s; check `docker compose logs neo4j` |
| Backend is slow to start | It is geocoding countries at ~1 req/s on startup — see [Start the backend](#3-start-the-backend) |
| Upload fails or times out | Use `python ingest.py` for large files; check that the sender/receiver/amount/timestamp columns exist and that amount is numeric |
| Dashboard says "No dataset selected" | Open a dataset from the Datasets page (**ANALYZE**) |
| 0 flagged transactions | Lower the threshold slider; the model is trained on IBM-style data, so other formats may score low |

## Repository layout

```
├── backend/
│   ├── main.py                  FastAPI app and all endpoints
│   ├── ingest.py                Bulk-load script for large datasets
│   ├── create_10k_sample.py     Cut a 10k-row sample from HI-Small_Trans.csv
│   ├── requirements.txt
│   ├── data/                    Reference data (CSVs git-ignored)
│   └── ml/                      Model definition, inference handler, weights, scalers
├── frontend/
│   ├── public/intro.html        Standalone cinematic intro page
│   └── src/                     App shell, api.js, and pages/ (Datasets, Dashboard, Investigate)
├── final-docs/                  SRS, testing report, user manual, technical-debt plan, etc.
├── documentation.md             Extended project and demo notes
└── docker-compose.yml           Neo4j container
```

## Documentation

- [final-docs/User_Manual.md](final-docs/User_Manual.md) — end-user guide
- [final-docs/SRS.md](final-docs/SRS.md) — software requirements specification
- [final-docs/Project_Documentation.md](final-docs/Project_Documentation.md) — consolidated project documentation
- [final-docs/Testing_Report.md](final-docs/Testing_Report.md) — test cases and results
- [final-docs/Technical_Debt_Plan.md](final-docs/Technical_Debt_Plan.md) — known issues and remediation plan
- [documentation.md](documentation.md) — extended walkthrough and demo talking points

## License

MIT — see [LICENSE](LICENSE).

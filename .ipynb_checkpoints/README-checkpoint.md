# Customer Segmentation — Marketing Campaign Analysis

2,239 customers. One marketing strategy. That's the problem.

This project groups customers by actual behaviour — what they buy, how often,
how they respond to campaigns — then serves those segments through an API
and a Streamlit dashboard the marketing team can use directly.

**Live demo:** [https://customer-segmentation-iq.streamlit.app/]
**API:** [https://customersegmentation-production-4685.up.railway.app]


## What I Found

| Segment | Customers | Avg Spend | Avg Income | Avg Purchases |
|---|---|---|---|---|
| High Value Champions | 770 (34%) | $1,223 | $73,902 | 19 |
| Budget Shoppers | 1,008 (45%) | $681 | $53,164 | 16 |
| Occasional Browsers | 461 (21%) | $98 | $34,562 | 6 |

34% of customers drive the majority of revenue.
The other 66% need a completely different approach — or you're spending
campaign budget where it won't convert.

## How I Built It

**Data Cleaning first**

Three issues in the raw data that would have broken clustering silently:
- Marital status had entries like `YOLO` and `Absurd` — mapped to `Single`
- Income had ~1% missing — filled with median, not mean, because income is right-skewed
- A handful of ages above 90 — removed as entry errors, not real customers

**Feature engineering**

Raw columns were too granular. I collapsed them into six signals
that actually describe how a customer behaves:

| Feature | What it captures |
|---|---|
| `TotalSpend` | Spend across all product categories |
| `TotalPurchases` | Purchases across all channels |
| `TotalCampaignsAccepted` | How often they respond to campaigns |
| `CustomerTenure` | Days since joining |
| `EducationLevel` | Ordinal encoding (1–4) |
| `Age` | Derived dynamically from birth year |

**Feature selection**

I measured variance across all numeric columns and kept the top 8.
Low-variance columns look almost identical across customers —
they add noise without helping the model separate anyone.

Selected: `NumDealsPurchases`, `NumWebVisitsMonth`, `TotalPurchases`,
`Age`, `Recency`, `CustomerTenure`, `TotalSpend`, `Income`

**Picking K**

Tested K=2 through K=12 using inertia and silhouette score together.
K=2 peaked on silhouette but the groups were too broad to act on.
K=3 gave a score of ~0.35 with three segments that are genuinely
different in ways a marketing team can do something with.

Everything — scaling and model — lives inside a single sklearn Pipeline
so new customers get transformed the same way at prediction time.

**Confirming the clusters**

PCA compressed the 8 features to 2 dimensions to visually check
the clusters were actually distinct and not overlapping noise.


## Project Structure

```
CUSTOMER_SEGMENTATION/
├── data/
│   └── MARKETING CAMPAIGN DATA.csv
├── scripts/
│   ├── 01_data_cleaning.py
│   ├── 02_feature_engineering.py
│   ├── 03_feature_selection.py
│   ├── 04_model_training.py
│   └── 05_segment_analysis.py
├── models/
│   └── market_segmentation_model.pkl
├── outputs/
│   ├── data_cleaned.csv
│   ├── data_engineered.csv
│   ├── data_model_ready.csv
│   ├── data_with_labels.csv
│   └── *.png / *.html
├── app.py
├── main.py
├── run_pipeline.py
└── requirements.txt
-----market_clustering_analysis.ipynb
-----ReadMe.md

```


## Running It

**Install**
```bash
git clone https://github.com/GODFREY-PNG/customer_segmentation.git
cd customer_segmentation
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

**Run the full pipeline**
```bash
python run_pipeline.py
```
Runs all five scripts in order. Saves cleaned data, plots, and the
trained model to their folders automatically.

**Start the API**
```bash
uvicorn main:app --reload
```
Runs at `http://127.0.0.1:8000` · Docs at `/docs`

**Start the dashboard**
```bash
streamlit run app.py
```
Runs at `http://localhost:8501`


## API

**`POST /predict`** — takes a customer profile, returns their segment,
a description, and a recommended campaign strategy.

```bash
curl -X POST http://127.0.0.1:8000/predict \
  -H "Content-Type: application/json" \
  -d '{
    "income": 75000,
    "age": 45,
    "recency": 10,
    "total_spend": 1200,
    "total_purchases": 20,
    "num_web_visits_month": 3,
    "num_deals_purchases": 1,
    "customer_tenure": 1200
  }'
```

**`GET /analytics`** — returns full segment stats for the dashboard.


## Stack

Python · Pandas · Scikit-learn · FastAPI · Uvicorn · Streamlit · Plotly · Seaborn


## Honest Limitations

Silhouette score of ~0.35 is reasonable for real customer data
but the cluster boundaries aren't sharp. Three things worth exploring:

- DBSCAN or Gaussian Mixture Models to handle the overlap differently
- A `/retrain` endpoint so the model updates as new campaign data arrives
- Docker setup to make deployment environment-independent
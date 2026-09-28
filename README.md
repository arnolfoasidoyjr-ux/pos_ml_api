# 🛒 POS ML API

Flask API serving **ARIMA sales forecasts** (daily/weekly/monthly per store) and **Apriori market basket recommendations** for the Pangga Store POS system.

- **Live URL:** `https://pos-ml-api.onrender.com`
- **Models trained:** 21 ARIMA models (one per store) + Apriori association rules
- **Model storage:** Hugging Face Hub → `https://huggingface.co/YOUR_HF_USERNAME/pos-ml-models`
- **Hosting:** Render (free tier)
- **Consumed by:** `ml_api.php` in the main POS repo

---

## 📡 Endpoints

### `GET /`
Health check.

    curl https://pos-ml-api.onrender.com/

```json
{
  "status": "ok",
  "message": "POS ML API is running",
  "models": { "arima": "21 store models available", "apriori": "142 rules loaded" },
  "hf_repo": "YOUR_HF_USERNAME/pos-ml-models",
  "version": "1.0.0"
}
```

---

### `GET /stores`
List every store that has a trained ARIMA model, with its accuracy metrics.

    curl https://pos-ml-api.onrender.com/stores

```json
{
  "stores": [
    { "store_id": "BAR-01", "mae": 145.2, "rmse": 210.5, "mape": 8.3 },
    { "store_id": "BAR-02", "mae": 132.7, "rmse": 198.1, "mape": 7.9 }
  ]
}
```

---

### `POST /forecast`
30-day (or custom) sales forecast for a single store.

    curl -X POST https://pos-ml-api.onrender.com/forecast \
      -H "Content-Type: application/json" \
      -d '{"store_id": "BAR-01", "forecast_days": 30}'

| Field | Type | Required | Notes |
|---|---|---|---|
| `store_id` | string | ✅ | Must match a store returned by `/stores` |
| `forecast_days` | int | ❌ | Default `30`, max `90` |

```json
{
  "store_id": "BAR-01",
  "forecast": [
    { "date": "2026-07-04", "predicted_sales": 772.08, "lower_bound": 540.00, "upper_bound": 1010.00 }
  ]
}
```

---

### `POST /recommend`
Apriori "customers also buy" recommendations based on current cart contents.

    curl -X POST https://pos-ml-api.onrender.com/recommend \
      -H "Content-Type: application/json" \
      -d '{"cart_items": ["Ganador Sardines in Tomato Sauce 155g"], "top_n": 5}'

| Field | Type | Required | Notes |
|---|---|---|---|
| `cart_items` | array of strings | ✅ | Exact product names as trained |
| `top_n` | int | ❌ | Default `5` |

```json
{
  "cart_items": ["Ganador Sardines in Tomato Sauce 155g"],
  "recommendations": [
    { "recommended_products": "Premium White Rice 5kg", "confidence": 0.45, "lift": 2.1, "support": 0.03 }
  ]
}
```

---

### `GET /metrics`
Training performance summary for all models.

    curl https://pos-ml-api.onrender.com/metrics

```json
{
  "arima_metrics": { "BAR-01": {"mae": 145.2, "rmse": 210.5, "mape": 8.3} },
  "apriori_summary": { "total_rules": 142, "min_support": 0.02, "min_confidence": 0.30 },
  "trained_on": "2026-06-15"
}
```

---

## 🧠 How Model Storage Works

Models are **not** stored in this repo (GitHub rejects large `.pkl` files). Instead:

1. Colab trains the models → uploads them to **Hugging Face Hub**
2. This repo only contains **code** (`app.py`, `requirements.txt`, etc.)
3. On first request per store, `app.py` downloads the needed `.pkl` from Hugging Face and caches it locally on the Render instance (`model_cache/`)
4. Every request after that uses the cache — no re-download until the instance restarts

```
Colab (train) → Hugging Face (store .pkl/.csv/.json) → Render (downloads + caches on demand) → PHP POS (calls the API)
```

---

## ⚙️ Environment Variables (set in Render dashboard)

| Key | Required | Example |
|---|---|---|
| `HF_REPO_ID` | ✅ | `YOUR_HF_USERNAME/pos-ml-models` |
| `HF_TOKEN` | Only if HF repo is **private** | `hf_xxxxxxxxxxxx` |
| `PYTHON_VERSION` | ✅ | `3.11.0` |

---

## 🔁 Retraining & Updating Models

1. Re-run `ML_Training_POS.ipynb` in Colab with new data
2. Run the "Upload to Hugging Face" cell at the end — it overwrites the existing files on HF
3. That's it. Render will pick up the new models automatically the next time each store's forecast is requested (or force it sooner with **Manual Deploy** on Render)

No code changes or redeploys needed for a retrain — only for actual API logic changes.

---

## 🐘 Using From PHP (`ml_api.php`)

```php
require_once 'ml_api.php';
$ml = new MLApi();

// Forecast
$result = $ml->forecast('BAR-01', 30);

// Recommend
$result = $ml->recommend(['Sardines 155g', 'Rice 5kg'], 5);
```

`ml_api.php` also works as a direct AJAX endpoint:

    ml_api.php?action=forecast&store_id=BAR-01&days=30
    ml_api.php?action=recommend   (POST, JSON body: {"cart_items": [...]})
    ml_api.php?action=stores
    ml_api.php?action=metrics

---

## 🩺 Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| First request after idle is slow (~30s) | Render free tier cold start | Normal — upgrade to a paid plan for always-on, or ping with UptimeRobot |
| `/stores` returns only 1 store | Cold-start timeout cached a fallback response | Wait for the 1-hour cache to expire, or hit `?refresh=1` from the POS dropdown's 🔄 button |
| `404` on `/forecast` | `store_id` doesn't match any trained model | Check `/stores` for the exact valid IDs |
| `503` on `/recommend` | Apriori rules failed to load from HF | Check `apriori_rules.csv` exists in the HF repo |
| Render build fails | Missing dependency | Confirm `requirements.txt` includes `huggingface_hub` |

---

## 📁 Repo Structure

```
pos-ml-api/
├── app.py
├── requirements.txt
├── Procfile
├── render.yaml
├── .gitignore
└── ml_api.php
```

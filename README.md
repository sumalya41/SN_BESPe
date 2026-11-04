Here is a professional, comprehensive `README.md` file tailored for your GitHub repository. It highlights the dataset's structure, the instruments covered, the schema, and includes the memory-efficient Python code we developed to handle the large-scale Parquet files.

***

```markdown
# 📊 SN-BES: High-Frequency Crypto Market Microstructure Dataset

Welcome to the **SN-BES** repository, a comprehensive, high-frequency cryptocurrency market microstructure dataset designed for quantitative research, algorithmic trading, and machine learning. 

This dataset provides granular, tick-by-tick data including Limit Order Book (LOB) snapshots, Best Bid/Offer (BBO), trade ticks, and pre-computed ML target variables across multiple spot and perpetual futures markets.

---

## 🌟 Key Features

- **Ultra-High Frequency**: Millisecond-level timestamp resolution.
- **Multi-Asset & Multi-Venue**: Covers major assets (BTC, ETH, SOL, PEPE) across USDT, USDC, FDUSD, and USD1 pairs.
- **Comprehensive Data Types**: Includes BBO, full Order Book depth, raw trades, and ML targets.
- **Optimized Storage**: Stored in highly compressed, columnar **Parquet** format, partitioned by date.
- **Built-in Metadata**: Includes daily data quality metrics, tick sizes, and target bucket edges.

---

## 📂 Dataset Structure

The dataset is organized by instrument and data type, partitioned by date (`YYYY-MM-DD.parquet`).

```text
sn-bes/
├── data-1000PEPEUSDC_PERP/
│   └── data/
│       ├── bbo/1000PEPEUSDC_PERP/
│       ├── book/1000PEPEUSDC_PERP/
│       └── trades/1000PEPEUSDC_PERP/
├── data-BTCUSDT_PERP/
│   └── data/
│       ├── bbo/BTCUSDT_PERP/
│       ├── book/BTCUSDT_PERP/
│       ├── targets/BTCUSDT_PERP/   <-- ML Target Variables
│       └── trades/BTCUSDT_PERP/
├── data-meta/                       <-- Data Quality & Instrument Metadata
│   └── data/
│       ├── meta/daily/
│       └── stable/
└── [Other Instruments...]
```

### Data Categories
| Folder | Description |
| :--- | :--- |
| **`bbo/`** | Best Bid and Offer (Top of Book) price and quantity updates. |
| **`book/`** | Limit Order Book (LOB) depth snapshots (L2/L3). |
| **`trades/`** | Tick-by-tick executed trades (price, size, side). |
| **`targets/`** | Pre-computed forward-looking returns/buckets for ML models. |
| **`meta/`** | Instrument specifications, tick sizes, and daily data quality reports. |

---

## 🪙 Supported Instruments (23 Total)

The dataset covers **4 base assets** across **4 quote currencies** in both **Spot** and **Perpetual** markets.

### 🟠 Bitcoin (BTC)
- `BTCUSDT_SPOT`, `BTCUSDT_PERP`
- `BTCUSDC_SPOT`, `BTCFDUSD_SPOT`, `BTCUSD1_SPOT`

### 🔵 Ethereum (ETH)
- `ETHUSDT_SPOT`, `ETHUSDT_PERP`
- `ETHUSDC_SPOT`, `ETHFDUSD_SPOT`, `ETHUSD1_SPOT`

### 🟣 Solana (SOL)
- `SOLUSDT_SPOT`, `SOLUSDT_PERP`
- `SOLUSDC_SPOT`, `SOLFDUSD_SPOT`

### 🐸 Pepe (PEPE)
- `PEPEUSDT_SPOT`
- `1000PEPEUSDT_PERP`, `1000PEPEUSDC_PERP`

*(Note: Stablecoin reference pairs like `USDCUSDT`, `FDUSDUSDT`, and `USD1USDT` are also included in the metadata for cross-conversion and peg tracking).*

---

## 📐 Data Schema

### BBO (Best Bid/Offer) Schema
| Column | Type | Description |
| :--- | :--- | :--- |
| `ts` | `int64` | Unix timestamp in milliseconds. |
| `bid_px` | `double` | Best bid price. |
| `ask_px` | `double` | Best ask price. |
| `bid_qty` | `float` | Best bid quantity. |
| `ask_qty` | `float` | Best ask quantity. |

### Metadata (`meta/daily`) Schema
Used to track data quality and instrument specifications per day.
| Column | Type | Description |
| :--- | :--- | :--- |
| `instrument` | `string` | The trading pair symbol. |
| `date` | `string` | Date of the snapshot (`YYYY-MM-DD`). |
| `tick_size` | `double` | Minimum price increment for the instrument. |
| `bucket_edges_bps` | `list/double` | Basis point thresholds for ML target bucketing. |
| `top_mismatch` | `double` | Data quality metric (BBO mismatch/anomaly rate). |
| `bytes` | `int64` | Uncompressed size of the raw data for the day. |
| `build_seconds`| `int64` | Time taken to process/validate the dataset. |

---

## 🚀 Getting Started

### ⚠️ Important: Memory Management
Because this dataset contains thousands of daily Parquet files, **do not** use `pd.concat()` or load all files into a list at once, as this will cause Out-Of-Memory (OOM) errors. 

Always use **PyArrow** to read only the columns you need, and process files iteratively.

### Prerequisites
```bash
pip install pandas pyarrow
```

### 1. Reading a Single BBO File
```python
import pandas as pd

# Load a single day of BBO data
bbo_df = pd.read_parquet(
    "data-BTCUSDT_PERP/data/bbo/BTCUSDT_PERP/2025-09-15.parquet",
    columns=['ts', 'bid_px', 'ask_px'] # Only load what you need!
)

# Calculate mid-price and spread
bbo_df['mid_px'] = (bbo_df['bid_px'] + bbo_df['ask_px']) / 2
bbo_df['spread_bps'] = ((bbo_df['ask_px'] - bbo_df['bid_px']) / bbo_df['mid_px']) * 10000

print(bbo_df.head())
```

### 2. Memory-Efficient Metadata Extraction
To extract a unified report of all instruments without crashing your RAM, use this iterative PyArrow approach:

```python
import pyarrow.parquet as pq
import pandas as pd
from pathlib import Path

base_path = Path("data-meta/data")
latest_meta = {}

for file_path in base_path.rglob("*.parquet"):
    try:
        pf = pq.ParquetFile(file_path)
        # Read only necessary columns to save RAM
        df = pf.read(columns=['instrument', 'date', 'tick_size', 'top_mismatch']).to_pandas()
        
        for _, row in df.iterrows():
            inst = row['instrument']
            if inst not in latest_meta or str(row['date']) > str(latest_meta[inst]['date']):
                latest_meta[inst] = row.to_dict()
    except Exception:
        continue

# Create a clean summary DataFrame
meta_df = pd.DataFrame(list(latest_meta.values())).sort_values('instrument')
print(meta_df)
```

---

## 🧠 Potential Use Cases

1. **Market Microstructure Analysis**: Study order book imbalance, price impact, and spread dynamics.
2. **High-Frequency Trading (HFT)**: Develop and backtest market-making or statistical arbitrage strategies.
3. **Machine Learning**: Train LSTM, Transformer, or XGBoost models using the `targets/` folder to predict short-term price movements.
4. **Cross-Exchange Arbitrage**: Analyze price discrepancies between Spot and Perpetual markets (e.g., `BTCUSDT_SPOT` vs `BTCUSDT_PERP`).

---

## 📜 License & Citation

This dataset is provided for research and educational purposes. 

If you use this dataset in your research, please cite this repository:
```bibtex
@misc{sn-bes-dataset-2026,
  author = {Your Name/Team},
  title = {SN-BES: High-Frequency Crypto Market Microstructure Dataset},
  year = {2026},
  publisher = {GitHub},
  howpublished = {\url{https://github.com/yourusername/sn-bes}}
}
```

---
*Built with ❤️ for the Quantitative Finance & Crypto Research Community.*
```

### 💡 Tips for your GitHub Repo:
1. **Replace Placeholders**: Make sure to update the `yourusername` in the citation block at the bottom.
2. **Add a `.gitignore`**: Since this is a data-heavy project, ensure you add `*.parquet`, `*.csv`, and `__pycache__/` to your `.gitignore` so you don't accidentally commit the massive data files to GitHub! Only commit the code and the README.
3. **Kaggle Integration**: If this is linked to a Kaggle dataset, you can add a badge at the very top of the README: `[![Kaggle](https://kaggle.com/static/images/open-in-kaggle.svg)](https://kaggle.com/datasets/your-dataset-link)`

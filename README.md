# data-highway

A statistical study of how indexing and configuration affect query execution time in PostgreSQL. The project runs controlled experiments on medium and large datasets, measures each query with `EXPLAIN ANALYZE`, and analyses the results with descriptive statistics, ANOVA, Tukey HSD, t tests and regression.

This is not a generic optimizer. It is a reproducible harness for testing indexing strategies under different memory and parallelism settings, with plots and inferential results checked into the repo.

## Problem

Query plans change with data size, index choice and memory settings. Teams often add indexes without measuring. This project asks which index types actually reduce execution time for which query patterns, and whether more memory helps or is wasted.

## Method

The harness applies one configuration and one index strategy at a time, then times each query 35 times with cache cleared between runs. Execution time comes from `EXPLAIN ANALYZE`. Results are grouped by query and index strategy for analysis.

Configurations tested
- Baseline: `shared_buffers 1GB`, `work_mem 16MB`, `effective_cache_size 4GB`
- High Memory: `shared_buffers 2GB`, `work_mem 32MB`, `effective_cache_size 6GB`

See `Code/Dataset Generation/configurations.py:1`.

## Queries

Seven queries cover common patterns. See `Code/Dataset Generation/queries.py:1`.

- `query_1` filter on `customers.created_date`
- `query_2` filter on `products.category`
- `query_3` group by `products.category` with average price
- `query_4` join `orders` and `customers` filtered by email
- `query_5` join `orders`, `order_items`, `products` with date range
- `query_6` join `products` and `order_items` with having and order by
- `query_7` left join to find products never ordered

## Indexing strategies

Defined in `Code/Dataset Generation/indexes.py:1`.

- `no_indexes` no user indexes, baseline
- `single_column_indexes` three to four single column indexes
- `composite_indexes` three composite indexes on join keys
- `partial_indexes` two partial indexes with where clauses
- `expression_indexes` indexes on `LOWER()` and concatenated columns
- `covering_indexes` two indexes with `INCLUDE` columns

Each strategy is created, measured, then dropped before the next run. See `Code/Dataset Generation/experiment_runner.py:144`.

## Datasets

Two scale factors with the same schema

- Medium dataset: baseline and high memory runs
- Large dataset: baseline and high memory runs

CSVs are in `Datasets/`. Each file records `Query`, `RunNumber`, `ExecutionTime_ms`, `Configuration`, `IndexStrategy`, `DataSetSize`. Four files: `medium_baseline.csv`, `medium_high.csv`, `large_baseline.csv`, `large_high.csv`.

## Experimental procedure

Implemented in `Code/Dataset Generation/experiment_runner.py:1`.

1. Vacuum the database
2. Apply configuration with `ALTER SYSTEM SET` and restart PostgreSQL
3. Drop all non system indexes and create the strategy under test
4. For each query, run `EXPLAIN ANALYZE` 35 times with `clear_cache()` between runs
5. Parse `Execution Time` from the plan output and append to CSV
6. Repeat for every configuration and strategy combination

Cache clearing uses `pg_stat_reset`, `DISCARD ALL` and dropping OS caches on Linux. See `clear_cache:46`.

## Analysis

Implemented in `Code/Analysis/analysis.py:1`. For each dataset file

- Descriptive statistics grouped by query and strategy, mean, median, std, variance, min, max, count
- Box plots of execution time distribution
- One way ANOVA per query `ExecutionTime_ms ~ C(IndexStrategy)`
- Tukey HSD pairwise comparisons per query with simultaneous confidence plots
- Pairwise Welch t tests between strategies per query
- Regression `ExecutionTime_ms ~ C(IndexStrategy) + C(Query)` with residuals vs fitted, Q Q plot and histogram of residuals
- Correlation between execution time and run number

Outputs go to `Inferential Results/results/` as CSV and text, and `Inferential Results/plots/` as PNG. Example files are `medium_baseline_descriptive_statistics.csv` and `large_high_regression_summary.txt`. Summary plots are also duplicated under `Descriptive Plots/` for quick viewing.

See `Inferential Results/results/large_baseline_descriptive_statistics.csv:1` for a sample of numeric output.

## Results

The plots and tables are versioned, so read them directly. In general, single column and composite indexes help filtered and joined queries, covering indexes help queries that can be satisfied from the index, and expression indexes matter only when the query uses the same expression. High memory reduces variance but does not replace correct indexing. Full numeric results are in `Inferential Results/` and the detailed narrative is in `Optimization of Database Query Performance.docx`.

A visual summary is in `Descriptive Plots/Medium Dataset/` and `Descriptive Plots/Large Dataset/` with histogram, boxplot, scatterplot, lineplot, heatmap and barplot per configuration.

## Project structure

```text
.
├── Code
│   ├── Analysis
│   │   └── analysis.py              # ANOVA, Tukey, t tests, regression
│   └── Dataset Generation
│       ├── experiment_runner.py     # runs EXPLAIN ANALYZE 35 times per query
│       ├── queries.py               # seven test queries
│       ├── indexes.py               # six index strategies
│       ├── configurations.py        # baseline vs high memory
│       └── plots.py                 # descriptive plots
├── Datasets
│   ├── medium_baseline.csv
│   ├── medium_high.csv
│   ├── large_baseline.csv
│   └── large_high.csv
├── Descriptive Plots
│   ├── Medium Dataset
│   └── Large Dataset
├── Inferential Results
│   ├── results                      # CSV and txt from analysis.py
│   └── plots                        # PNG from analysis.py
├── Optimization of Database Query Performance.docx
└── README.md
```

## Installation

Prerequisites are Python 3.10 or later and a local PostgreSQL 17 instance. The experiment runner expects `DB_PARAMS` in `experiment_runner.py:11` to point to your database and adjusts `POSTGRES_BIN_PATH` and `POSTGRES_DATA_PATH` for your OS.

```bash
git clone https://github.com/mastaan66/data-highway.git
cd data-highway

python -m venv .venv
source .venv/bin/activate

pip install -r requirements.txt
```

If `requirements.txt` is missing, install the analysis dependencies directly

```bash
pip install pandas numpy matplotlib seaborn statsmodels scipy psycopg2-binary
```

## Usage

Run the full experiment matrix. This will write `experiment_results.csv` in the current directory.

```bash
python "Code/Dataset Generation/experiment_runner.py"
```

Generate descriptive plots

```bash
python "Code/Dataset Generation/plots.py"
```

Run inferential analysis. Place the four dataset CSVs in the working directory first, then

```bash
python Code/Analysis/analysis.py
```

Outputs appear in `results/` and `plots/` under the working directory. The repo already contains a completed run in `Datasets/` and `Inferential Results/`.

Example of querying the results programmatically

```python
import pandas as pd

df = pd.read_csv("Datasets/medium_baseline.csv")
summary = df.groupby(["Query", "IndexStrategy"])["ExecutionTime_ms"].mean()
print(summary.head())
```

## Reproducing the paper

The Word document is the full writeup with introduction, methodology, results and discussion. Plots referenced in the paper are the same PNGs in `Descriptive Plots/` and `Inferential Results/plots/`. To rebuild everything from scratch, clean the database, run the experiment runner, then run the plots and analysis scripts in order.

## Limitations

- Timings depend on hardware, shared buffers and concurrent load, so absolute milliseconds do not transfer across machines
- The harness measures server side execution time only, not network or client parsing
- Expression and covering indexes are tested with the specific expressions in this schema, not as a general rule

## Contributing

Issues and pull requests are welcome. Keep changes focused and include before and after numbers when you adjust queries, indexes or configurations.

## License

MIT. See `LICENSE`.

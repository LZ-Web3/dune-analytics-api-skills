# Table Discovery

Find and inspect Dune tables programmatically before writing SQL.

## Table of Contents
1. [Get Table Schema](#get-table-schema)
2. [Search Tables by Name](#search-tables-by-name)
3. [List Schemas](#list-schemas)
4. [List User Uploads](#list-user-uploads)
5. [Common Discovery Patterns](#common-discovery-patterns)

---

## Get Table Schema

Inspect columns, types, and nullability of any table using its slug (e.g., `dex.trades`, `ethereum.transactions`).

```python
from dune_client.client import DuneClient
import os

client = DuneClient(api_key=os.environ['DUNE_API_KEY'])

# Get schema for a curated table
result = client.get_dataset("dex.trades")
print(f"Table: {result.full_name}")
print(f"Type: {result.type}")
for col in result.columns:
    print(f"  {col.name}: {col.type} (nullable={col.nullable})")
```

**Supported slugs:** Use `schema.table` format:
- Curated: `dex.trades`, `dex_aggregator.trades`, `dex_solana.trades`, `nft.trades`, `tokens.erc20`, `prices.usd`
- Raw: `ethereum.transactions`, `solana.transactions`, `polygon.logs`
- User uploads: `dune.<namespace>.dataset_<table_name>`

## Search Tables by Name

Use `information_schema.tables` via SQL to search for tables by keyword:

```python
# Search for tables containing a keyword
result = client.run_sql(
    query_sql="""
    SELECT table_schema, table_name
    FROM information_schema.tables
    WHERE CAST(table_schema AS VARCHAR) LIKE '%keyword%'
       OR CAST(table_name AS VARCHAR) LIKE '%keyword%'
    LIMIT 20
    """,
    performance='medium',
    ping_frequency=5
)
for row in result.result.rows:
    print(f"{row['table_schema']}.{row['table_name']}")
```

**⚠️ Notes:**
- `information_schema` queries consume credits (uses `run_sql`, Plus plan required)
- Cast columns to VARCHAR for LIKE comparisons
- Use specific filters to avoid scanning millions of decoded tables

### Targeted Search Examples

```python
# Find all event tables for a protocol on a specific chain
result = client.run_sql(
    query_sql="""
    SELECT table_schema, table_name
    FROM information_schema.tables
    WHERE CAST(table_schema AS VARCHAR) = 'uniswap_v3_ethereum'
      AND CAST(table_name AS VARCHAR) LIKE 'evt_%'
    LIMIT 50
    """,
    performance='medium',
    ping_frequency=5
)

# Find all tables related to a token/project
result = client.run_sql(
    query_sql="""
    SELECT table_schema, table_name
    FROM information_schema.tables
    WHERE CAST(table_name AS VARCHAR) LIKE '%swap%'
      AND CAST(table_schema AS VARCHAR) LIKE '%ethereum%'
    LIMIT 20
    """,
    performance='medium',
    ping_frequency=5
)
```

## List Schemas

Find available schemas (protocol + chain namespaces):

```python
# List all schemas matching a keyword
result = client.run_sql(
    query_sql="SHOW SCHEMAS LIKE '%uniswap%'",
    performance='medium',
    ping_frequency=5
)
for row in result.result.rows:
    print(row['Schema'])
```

**Schema naming conventions:**
- Decoded tables: `<protocol>_<chain>` (e.g., `uniswap_v3_ethereum`, `aave_v3_polygon`)
- Raw tables: `<chain>` (e.g., `ethereum`, `solana`, `polygon`)
- Curated: `dex`, `dex_solana`, `nft`, `tokens`, `prices`, `labels`
- User uploads: `dune.<namespace>`

## List User Uploads

List tables uploaded via CSV or create_table API:

```python
result = client.list_uploads(limit=50)
for table in result.tables:
    print(f"{table.full_name} ({table.table_size_bytes} bytes)")
    for col in table.columns:
        print(f"  - {col.name}: {col.type}")
```

## Common Discovery Patterns

### 1. "What tables exist for protocol X on chain Y?"

```python
# Step 1: Find the schema
result = client.run_sql(
    query_sql="SHOW SCHEMAS LIKE '%aave%ethereum%'",
    performance='medium', ping_frequency=5
)
# → aave_v3_ethereum, aave_v2_ethereum, ...

# Step 2: List tables in that schema
result = client.run_sql(
    query_sql="""
    SELECT table_name FROM information_schema.tables
    WHERE CAST(table_schema AS VARCHAR) = 'aave_v3_ethereum'
    LIMIT 50
    """,
    performance='medium', ping_frequency=5
)

# Step 3: Inspect a specific table
schema = client.get_dataset("aave_v3_ethereum.evt_Supply")
```

### 2. "What columns does this table have?"

```python
result = client.get_dataset("dex_aggregator.trades")
for col in result.columns:
    print(f"{col.name}: {col.type}")
```

### 3. "What curated/spell tables are available?"

```python
result = client.list_datasets(type="spell", limit=250)
for ds in result.datasets:
    print(f"{ds.full_name}")
```

Available types: `spell`, `decoded_table`, `uploaded_table`, `transformation_view`, `transformation_table`, `dune_table`

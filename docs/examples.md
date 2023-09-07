
# Examples

## Providing IOManagers to `Definitions`

```python
from dagster import Definitions
from dagster_polars import PolarsDeltaIOManager, PolarsParquetIOManager

base_dir = (
    "/remote/or/local/path"  # s3://my-bucket/... or gs://my-bucket/... also works!
)

definitions = Definitions(
    resources={
        "polars_parquet_io_manager": PolarsParquetIOManager(base_dir=base_dir),
        "polars_delta_io_manager": PolarsDeltaIOManager(base_dir=base_dir),
    }
)
```

## Reading specific columns
```python
import polars as pl
from dagster import AssetIn, asset


@asset(
    io_manager_key="polars_parquet_io_manager",
    ins={
        "upstream": AssetIn(metadata={"columns": ["a"]})
    },  # explicitly specify which columns to load
)
def downstream(upstream: pl.DataFrame):
    assert upstream.columns == ["a"]
```

## Reading `LazyFrame`

```python
import polars as pl
from dagster import asset


@asset(
    io_manager_key="polars_parquet_io_manager",
)
def downstream(
    upstream: pl.LazyFrame,  # the type annotation controls whether we load an eager or lazy DataFrame
) -> pl.DataFrame:
    df = ...  # some lazy operations with `upstream`
    return df.collect()
```

## Reading multiple partitions
```python
import polars as pl
from dagster import asset, StaticPartitionsDefinition
from dagster_polars import DataFramePartitions, LazyFramePartitions


@asset(
    partitions_def=StaticPartitionsDefinition(["a", "b"]),
    io_manager_key="polars_parquet_io_manager",
)
def upstream() -> pl.DataFrame:
    return pl.DataFrame(...)


@asset(
    io_manager_key="polars_parquet_io_manager",
)
def downstream_eager(upstream: DataFramePartitions):
    assert isinstance(upstream, dict)
    assert isinstance(upstream["a"], pl.DataFrame)
    assert isinstance(upstream["b"], pl.DataFrame)


@asset(
    io_manager_key="polars_parquet_io_manager",
)
def downstream_lazy(upstream: LazyFramePartitions):
    assert isinstance(upstream, dict)
    assert isinstance(upstream["a"], pl.LazyFrame)
    assert isinstance(upstream["b"], pl.LazyFrame)
```

## Skipping missing input/output

```python
from typing import Optional

import polars as pl
from dagster import asset


@asset(
    io_manager_key="polars_parquet_io_manager",
)
def downstream(upstream: Optional[pl.DataFrame]) -> Optional[pl.DataFrame]:
    maybe_df: Optional[pl.DataFrame] = ...
    return maybe_df
```

## Reading/writing custom metadata into storage

```python
import polars as pl
from dagster import asset
from dagster_polars import DataFrameWithMetadata


@asset(
    io_manager_key="polars_parquet_io_manager",
)
def upstream() -> DataFrameWithMetadata:
    return pl.DataFrame(...), {"my_custom_metadata": "my_custom_value"}


@asset(
    io_manager_key="polars_parquet_io_manager",
)
def downsteam(upstream: DataFrameWithMetadata):
    df, metadata = upstream
    assert metadata["my_custom_metadata"] == "my_custom_value"
```

## Append to DeltaLake table
```python
import polars as pl
from dagster import asset


@asset(io_manager_key="polars_parquet_io_manager")
def upstream() -> pl.DataFrame:
    return pl.DataFrame({"a": [1, 2, 3], "b": ["a", "b", "c"]})


@asset(
    io_manager_key="polars_delta_io_manager",
    metadata={
        "mode": "append"  # append to the existing table instead of overwriting it
    },
)
def downstream_append(upstream: pl.DataFrame) -> pl.DataFrame:
    return upstream
```
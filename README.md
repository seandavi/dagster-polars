# `dagster-polars`

[Polars](https://github.com/pola-rs/polars) integration library for [Dagster](https://github.com/dagster-io/dagster).

## Features
 - `BasePolarsUPathIOManager` is a base class for IO managers that work with Polars DataFrames. Shouldn't be used directly unless you want to implement your own `IOManager`.
   - returns the correct type (`polars.DataFrame` or `polars.LazyFrame`) based on the type annotation
   - logs various metadata about the DataFrame - size, schema, sample, stats, ...
   - the "columns" input metadata value can be used to select a subset of columns
   - inherits all the features of the `UPathIOManager` - works with local and remote filesystems (like S3),
       supports loading multiple partitions (use `dict[str, pl.DataFrame]` type annotation), ...
   - Implemented serialization formats:
     - `PolarsParquetIOManager` - for reading and writing files in Apache Parquet format. Supports reading partitioned Parquet datasets (for example, often produced by Spark).

## Quickstart

### Installation

```shell
pip install dagster-polars
```

### Usage
```python
import polars as pl
from dagster import asset, Definitions
from dagster_polars import PolarsParquetIOManager


@asset(io_manager_key="polars_parquet_io_manager")
def upstream() -> pl.DataFrame:
    df: pl.DataFrame = ...
    return df


@asset(io_manager_key="polars_parquet_io_manager")
def downstream(upstream: pl.LazyFrame) -> pl.DataFrame:
    df = ...  # some lazy operations with `upstream`
    return df.collect()


definitions = Definitions(
    assets=[upstream, downstream],
    resources={
        "polars_parquet_io_manager": PolarsParquetIOManager(base_dir="/remote/or/local/path")
    }
)
```

## Development

### Installation
```shell
poetry install
poetry run pre-commit install
```

### Testing
```shell
poetry run pytest
```

## TODO

 - [ ] Data validation like in [dagster-pandas](https://docs.dagster.io/integrations/pandas#validating-pandas-dataframes-with-dagster-types)
 - [ ] Maybe use `DagsterTypeLoader` ?

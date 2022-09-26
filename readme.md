# NYC Taxi Benchmark

## Overview

[cuDF](https://github.com/rapidsai/cudf) is NVIDIAs library that utilizes graphic computing to join, aggregate, filter, and manipulate data. cuDF is very similar to Pandas API and can almost be used as a drop-in replacement for Pandas.

This benchmark illustrates the potential speedup benefits of cuDF compared to Pandas on non-synthetic data.

## Data

The data is scraped from the [NYC Taxi and Limousine Commission website](https://www1.nyc.gov/site/tlc/about/tlc-trip-record-data.page) and contains the yellow taxi trip records. The total amount of rows is over 30 million entries. The column fields include pick-up and drop-off locations, dates and times, trip distances, passenger counts, trip distance, rate types, payment types, and itemized fares.

## Script

A script runs to download all Yellow Taxi Trips record from 2021:

```python
def download_files(year, download_directory):

    # Make directory if it does not exist
    pathlib.Path(download_directory).mkdir(parents=True, exist_ok=True)

    # Download yellow taxi data
    # Data from: https://www1.nyc.gov/site/tlc/about/tlc-trip-record-data.page
    url = "https://d37ci6vzurychx.cloudfront.net/trip-data/"
    files = [f"yellow_tripdata_{year}-{n:02d}.parquet" for n in range(1, 13)]
    for file in files:
        r = requests.get(f"{url}{file}", allow_redirects=True)
        with open(f"{download_directory}/{file}", "wb") as f:
            f.write(r.content)

    print("Finished downloading files...")


download_files(year, directory)
```

### RAPIDS cudf

```python
rapids_df = cudf.read_parquet(directory)
```

```python
rapids_df = rapids_df.query("fare_amount > 0 and tip_amount > 0 and passenger_count > 0 and trip_distance != 0")
rapids_df["tip_percentage"] = rapids_df["tip_amount"] / rapids_df["fare_amount"]
rapids_df["pickup_hour"] = rapids_df["tpep_pickup_datetime"].dt.hour
hours = rapids_df.groupby("pickup_hour", sort=True)["tip_percentage"].mean()
```

- Read Parquet Files: 1.62 seconds
- Data Transformations: 799 milliseconds
- Total Duration: 2.42 seconds

### Pandas

```python
pandas_df = pd.read_parquet(directory)
```

```python
pandas_df = pandas_df.query("fare_amount > 0 and tip_amount > 0 and passenger_count > 0 and trip_distance != 0")
pandas_df["tip_percentage"] = pandas_df["tip_amount"] / pandas_df["fare_amount"]
pandas_df["pickup_hour"] = pandas_df["tpep_pickup_datetime"].dt.hour
hours = pandas_df.groupby("passenger_count", sort=True)["tip_percentage"].mean()
```

- Read Parquet Files: 2.78 seconds
- Data Transformations: 4.34 seconds
- Total Duration: 7.12 seconds

## Results

|                          | **cuDF** | **Pandas** |
| ------------------------ | -------: | ---------: |
| **Read Parquet Files**   |   1.62 s |     2.78 s |
| **Data Transformations** |  0.799 s |     4.34 s |
| **Total Duration**       |   2.42 s |     7.12 s |

## Resources

- Data Source: [Yellow Trip Record Data](https://www1.nyc.gov/site/tlc/about/tlc-trip-record-data.page)
- Libraries: [`cuDF`](https://github.com/rapidsai/cudf), [`Pandas`](https://github.com/pandas-dev/pandas), [`Requests`](https://requests.readthedocs.io/en/latest/)
- Docker Container: [rapidsai:22.08-cuda11.5-runtime-ubuntu20.04-py3.9](http://nvcr.io/nvidia/rapidsai/rapidsai:22.08-cuda11.5-runtime-ubuntu20.04-py3.9)
- Source Code: [`benchmark.ipynb`](benchmark/benchmark.ipynb)

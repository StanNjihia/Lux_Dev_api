# Duka Connect: API Data Extraction & Transformation Pipeline

This project demonstrates an automated ETL (Extract, Transform, Load) pipeline built for Duka Connect,
a Nairobi-based B2B distributor. It extracts partner customer directories from a REST API, flattens
the complex nested JSON structures, and exports a clean, spreadsheet-ready CSV for the analytics team.

## Core Pipeline Features

* **Robust API Extraction:** Retrieves user data from
  <https://jsonplaceholder.typicode.com/users> utilizing `requests.get()`.
* **Advanced Error Handling:** Catches connection failures, timeouts, non-200 HTTP status codes, and
  invalid JSON bodies, raising a unified `FetchError` to prevent silent pipeline failures.
* **Dynamic JSON Flattening:** Uses a recursive function to automatically flatten nested dictionaries
  to any depth (e.g., transforming `address -> geo -> lat` into `address_geo_lat`) without
  hard-coding field names.
* **Data Cleaning & Standardization:** Uses Pandas to rename the `id` column to `customer_id`, convert
  geographical coordinates from strings to floats, and drop rows with duplicate or missing IDs.
* **Automated CSV Export:** Outputs a strictly formatted file to `output/customers_clean.csv`,
  filtering out any unexpected API columns to ensure a stable schema.

## Requirements

* Python 3.12 or higher.
* `pandas` for tabular data manipulation and CSV generation.
* `requests` for HTTP API calls.

Install everything (including Jupyter) with:

```bash
pip install -r requirements.txt
```

## Usage

To run the pipeline, execute the cells sequentially within the `extract_customers.ipynb` Jupyter
Notebook. The final execution cell contains a `main()` function that wires the extraction,
transformation, and export steps into a single automated sequence.

To run the whole notebook from the command line instead:

```bash
jupyter nbconvert --to notebook --execute --inplace extract_customers.ipynb
```

Re-running overwrites `output/customers_clean.csv` rather than appending to it.

## Environment Variables

* `CUSTOMERS_API_URL`: You can override the default API endpoint by setting this environment variable
  from the command line. This is primarily used to test the pipeline's failure handling against
  invalid hosts or 404 errors.

## Data Structure

* The API returns a **list** of 10 **dicts**, one per customer. Most values are strings; `id` is an int.
* **Nested fields:**
  * `address` (`street`, `suite`, `city`, `zipcode`): dict, **1 level** deep.
  * `address.geo` (`lat`, `lng`): dict inside `address`, **2 levels** deep.
  * `company` (`name`, `catchPhrase`, `bs`): dict, **1 level** deep.
* `lat` and `lng` arrive as **strings**, so they are converted to floats before export.

## Assumptions

* **Missing fields become empty cells (`None`).** Nested lookups use `.get()` with `{}` as the
  fallback, so a user with no `address` or `company` does not crash the pipeline. An empty cell was
  chosen over a placeholder such as `"Unknown"`, which could be mistaken for a real value.
* **Column names are lower_snake_case.** `company_catchPhrase` becomes `company_catchphrase` to match
  the expected output.
* **The schema is fixed.** Columns follow the order in `expected_customers_clean.csv`. New API fields
  are reported and left out; an expected field that is missing comes out empty.
* **One row per customer.** Rows with a blank or duplicate `customer_id` are removed, and the notebook
  prints how many were dropped.
* **Unparseable coordinates** become empty rather than stopping the run.

## Testing Failure Handling

Every failure prints one clear `ERROR:` message and stops with exit code 1, never a raw traceback.

* **Inside the notebook:** the Task 8 cell calls `fetch_users()` with four bad inputs. Output:

  ```text
  Wrong URL (host does not exist):
      Could not connect to https://jsonplaceholder.typicode.invalid/users. Check the URL and your internet connection.
  Non-200 response (404):
      https://jsonplaceholder.typicode.com/does-not-exist returned HTTP 404 (Not Found), expected 200.
  Body that is not JSON (HTML page):
      https://example.com returned a body that is not valid JSON (Content-Type: text/html; charset=utf-8).
  Timeout:
      Request to https://jsonplaceholder.typicode.com/users timed out after 0.001 seconds.
  ```

* **From the command line:** set `CUSTOMERS_API_URL` to a bad URL and run the notebook. In PowerShell:

  ```powershell
  $env:CUSTOMERS_API_URL = "https://jsonplaceholder.typicode.com/does-not-exist"
  jupyter nbconvert --to notebook --execute --stdout extract_customers.ipynb > $null
  $LASTEXITCODE
  Remove-Item Env:CUSTOMERS_API_URL
  ```

  | URL tested | Exit code | Message |
  |---|---|---|
  | `https://jsonplaceholder.typicode.invalid/users` | 1 | `ERROR: Could not connect to ...` |
  | `https://jsonplaceholder.typicode.com/does-not-exist` | 1 | `ERROR: ... returned HTTP 404 (Not Found), expected 200.` |
  | `https://example.com` | 1 | `ERROR: ... returned a body that is not valid JSON ...` |
  | default URL | 0 | run summary printed, CSV written |

  A failed run stops before any data is processed, so the CSV from the last good run is left untouched.

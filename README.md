# Renewable Energy Dashboard

Enhanced renewable-generation dashboard for a CS Capstone project.

Hosted at: http://pplrenewabledashboard.com/

## Project Structure

```text
Renewable-Energy-Dashboard/
|-- README.md                                  # Project setup, architecture, schema, and development notes.
|-- dashboard-scripts/                         # Active dashboard app.
|   |-- index.php                              # Main dashboard page and initial live data fetches.
|   |-- fetchData.php                          # Historical chart JSON endpoint.
|   |-- get_rolling_capacity_factors.php       # Latest rolling capacity factor JSON endpoint.
|   |-- wind_API.php                           # Server-side wrapper for the authenticated wind API.
|   |-- proxy.php                              # Proxy for browser-facing external JSON fetches.
|   |-- timeSeries.js                          # Highcharts setup, date range handling, and chart refresh logic.
|   |-- updateGauges.js                        # Live gauge and info-box refresh logic.
|   |-- rollingCapacityFactors.js              # Browser refresh logic for 7-day capacity factors.
|   |-- Wind_API.js                            # Browser refresh logic for wind-specific info boxes.
|   |-- gauge.js                               # Custom SVG gauge drawing helpers.
|   |-- style.css                              # Dashboard layout, responsive behavior, and visual styling.
|   |-- assets/images/                         # Favicon and image assets.
|   `-- config/                                # Local PHP config files plus examples.
|       |-- API/ConfigAPI.example.php          # Example wind API config.
|       `-- DB/dbConfig.example.php            # Example PHP database config.
|-- python-db-scripts/                         # Background data jobs.
|   |-- mysqldb_h_2.py                         # Inserts live samples into historical_data.
|   |-- calculate_rolling_capacity_factors.py  # Stores 7-day capacity factor snapshots.
|   |-- Inser_data_to_database.py              # One-off CSV import helper.
|   `-- config.example.py                      # Example Python config.
|-- dashboard-old-scripts/                     # Archived earlier dashboard implementation.
`-- dashboard-info/                            # Operational/deployment reference artifacts.
```

## System Architecture

The project uses a simple three-part architecture:

- **Browser dashboard:** `index.php` serves the initial HTML. Client-side JavaScript updates gauges, info boxes, rolling capacity factor fields, and the Highcharts time-series chart.
- **PHP endpoint layer:** PHP scripts read local config, call external APIs when needed, query MySQL, and return JSON to the browser. This keeps database credentials and API tokens out of browser code.
- **Python background jobs:** Python scripts run outside the request/response path. They collect live data into MySQL and precompute rolling capacity factor snapshots so the dashboard can read them quickly.

The database is the handoff point between the background jobs and the dashboard. Live values can be shown directly from external endpoints, but historical chart data and rolling capacity factors come from MySQL.

## Architecture Diagram

```text
                         External live data sources
                    +--------------------------------+
                    | ESS JSON endpoint              |
                    | Wind API endpoint              |
                    +---------------+----------------+
                                    |
              live fetches          |              scheduled collection
                                    |
+--------------------------+        |        +------------------------------+
| Browser                  |        |        | Python background jobs        |
| index.php + JavaScript   |        |        | mysqldb_h_2.py                |
|                          |        |        | calculate_rolling_capacity... |
+------------+-------------+        |        +--------------+---------------+
             |                      |                       |
             | fetch JSON           |                       | insert/query
             |                      |                       |
+------------v-------------+        |        +--------------v---------------+
| PHP endpoints            |<-------+        | MySQL database                |
| proxy.php                |                 | historical_data               |
| wind_API.php             |                 | capacity_factor_history       |
| fetchData.php            |<----------------+                              |
| get_rolling_capacity...  |                 +------------------------------+
+------------+-------------+
             |
             | chart data, gauge values, info-box values
             |
+------------v-------------+
| Dashboard UI             |
| Gauges, cards, chart     |
+--------------------------+
```

## Tech Stack

- **Frontend:** HTML, CSS, JavaScript, Highcharts Stock, and flatpickr.
- **Backend:** PHP 8+ for server-rendered markup and JSON endpoints.
- **Background jobs:** Python 3 for scheduled data collection and rolling capacity factor calculations.
- **Database:** MySQL 8.0 for historical samples and capacity factor snapshots.
- **External data sources:** ESS JSON endpoint and wind API.

PHP renders the initial page and handles server-side requests. After the page loads, JavaScript handles browser updates. The Python jobs must run separately to keep historical and rolling capacity factor data current.

## How The System Works

The dashboard combines live API data with historical data stored in MySQL.

- `dashboard-scripts/index.php` renders the page and does the initial live data fetch.
- `dashboard-scripts/updateGauges.js` refreshes live gauge and info-box values from the ESS JSON endpoint every 5 seconds.
- `dashboard-scripts/Wind_API.js` refreshes wind API info-box values through `wind_API.php` every 15 seconds.
- `dashboard-scripts/timeSeries.js` builds the Highcharts time-series chart and requests historical data from `fetchData.php`.
- `dashboard-scripts/fetchData.php` queries MySQL, groups records into 60-second intervals, and returns chart-ready JSON.
- `python-db-scripts/mysqldb_h_2.py` samples live percentage values every 5 seconds and writes them to `historical_data`.
- `python-db-scripts/calculate_rolling_capacity_factors.py` calculates 7-day rolling capacity factors every 5 minutes and writes them to `capacity_factor_history`.
- `dashboard-scripts/rollingCapacityFactors.js` fetches the latest rolling capacity factors every 5 minutes.

Timestamps are stored in the database in UTC. The dashboard accepts and displays dates in `America/New_York`, so PHP converts selected date ranges to UTC before querying MySQL and converts result timestamps back for chart display.

## Configuration

Sensitive config files are intentionally ignored by git. Create local copies from the committed examples:

```bash
cp dashboard-scripts/config/API/ConfigAPI.example.php dashboard-scripts/config/API/ConfigAPI.php
cp dashboard-scripts/config/DB/dbConfig.example.php dashboard-scripts/config/DB/dbConfig.php
cp python-db-scripts/config.example.py python-db-scripts/config.py
```

Then fill in the local values:

- `dashboard-scripts/config/API/ConfigAPI.php` needs `url` and `token` for the wind API.
- `dashboard-scripts/config/DB/dbConfig.php` needs `servername`, `username`, `password`, and `dbname` for the PHP endpoints.
- `python-db-scripts/config.py` needs the same local DB values for the Python jobs. It also contains optional CSV import and API variables used by helper scripts.

Do not commit real credentials or API tokens.

## Prerequisites

- PHP 8+ with `curl` enabled
- Python 3
- MySQL 8.0
- MySQL Workbench, or another MySQL client
- Python packages:

```bash
python3 -m pip install requests pymysql
```

## Database Setup

Create the database:

```sql
CREATE DATABASE renewables;
USE renewables;
```

Create `historical_data`:

```sql
CREATE TABLE historical_data (
    id INT NOT NULL AUTO_INCREMENT,
    date_time DATETIME NOT NULL,
    solar_percentage FLOAT DEFAULT NULL,
    wind_percentage FLOAT DEFAULT NULL,
    hydro_percentage FLOAT DEFAULT NULL,
    battery_percentage FLOAT DEFAULT NULL,
    solar_fixed_percentage FLOAT DEFAULT NULL,
    solar_360_percentage FLOAT DEFAULT NULL,
    electricity_demand FLOAT DEFAULT NULL,
    PRIMARY KEY (id, date_time),
    INDEX (date_time)
)
PARTITION BY RANGE (YEAR(date_time)) (
    PARTITION p2016 VALUES LESS THAN (2017),
    PARTITION p2017 VALUES LESS THAN (2018),
    PARTITION p2018 VALUES LESS THAN (2019),
    PARTITION p2019 VALUES LESS THAN (2020),
    PARTITION p2020 VALUES LESS THAN (2021),
    PARTITION p2021 VALUES LESS THAN (2022),
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION p2026 VALUES LESS THAN (2027),
    PARTITION p2027 VALUES LESS THAN (2028),
    PARTITION p2028 VALUES LESS THAN (2029),
    PARTITION p2029 VALUES LESS THAN (2030),
    PARTITION p2030 VALUES LESS THAN (2031)
);
```

Create `capacity_factor_history`:

```sql
CREATE TABLE capacity_factor_history (
    date_time DATETIME NOT NULL,
    solar_total_capacity_factor_7d FLOAT DEFAULT NULL,
    hydro_capacity_factor_7d FLOAT DEFAULT NULL,
    solar_fixed_capacity_factor_7d FLOAT DEFAULT NULL,
    solar_dual_capacity_factor_7d FLOAT DEFAULT NULL,
    wind_capacity_factor_7d FLOAT DEFAULT NULL,
    PRIMARY KEY (date_time)
) ENGINE=InnoDB;
```

These schemas match the dashboard and Python scripts. `historical_data.date_time` uses a composite primary key with `id` because the table is partitioned by year, while `capacity_factor_history.date_time` is the only primary key because each run stores one snapshot of the latest 7-day factors.

The yearly partitions on `historical_data` keep a continuously growing time-series table easier for MySQL to query and maintain. Most dashboard queries filter by date range, so partitioning by `YEAR(date_time)` lets MySQL narrow work to the relevant year partitions instead of scanning the entire history.

## Run Locally

Start the PHP server from the repo root:

```bash
php -S localhost:8000 -t dashboard-scripts
```

Start the historical data collector from `python-db-scripts/`:

```bash
cd python-db-scripts
python3 mysqldb_h_2.py
```

In another terminal, start the rolling capacity factor job from `python-db-scripts/`:

```bash
cd python-db-scripts
python3 calculate_rolling_capacity_factors.py
```

Let the Python jobs run for a few seconds, then confirm data is being inserted:

```sql
SELECT * FROM historical_data ORDER BY date_time DESC LIMIT 5;
SELECT * FROM capacity_factor_history ORDER BY date_time DESC LIMIT 5;
```

Open the dashboard:

```text
http://localhost:8000/index.php
```

## Implementation Notes

- Live percentages are clamped to `0-100` before they are stored or displayed.
- The default chart range is the most recent 36 hours.
- The date picker limits selected ranges to one month to keep database queries and chart rendering manageable.
- Historical chart data is grouped into 60-second intervals in `fetchData.php`.
- Electricity demand is derived from `CCL` with `sqrt(CCL) / 8549.9 * 100`.
- CO2 reduction is calculated from current solar, wind, hydro, and battery power values. Battery power is adjusted differently for charging and discharging.
- Wind lifetime capacity factor uses `2024-02-14 13:01:00 UTC` as the turbine online reference time and `90 kW` as the wind capacity.
- `python-db-scripts/Inser_data_to_database.py` is a one-off CSV import helper and is not required for normal dashboard operation.

## Troubleshooting

- If `index.php` fails immediately, check that `dashboard-scripts/config/API/ConfigAPI.php` exists and has a valid wind API URL and token.
- If the chart is empty, confirm that `python-db-scripts/mysqldb_h_2.py` is running and that `historical_data` has recent rows.
- If rolling capacity factors show `--%`, confirm that `python-db-scripts/calculate_rolling_capacity_factors.py` is running and that `capacity_factor_history` has rows.
- If local dates look shifted, check that database timestamps are being stored in UTC and that PHP is using `America/New_York` conversions.

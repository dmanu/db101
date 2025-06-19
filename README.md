## Databases 101 - Lab Content

### Prerequisites and Environment Setup (10 minutes)

**Required Files Setup**:

Before starting the lab, ensure you have the following files in your working directory:

- `docker-compose.yml` (provided in this repo)
- `taxi_trips.csv` (NYC taxi dataset. Partial is fine. e.g. https://github.com/DataTalksClub/nyc-tlc-data/releases/download/yellow/yellow_tripdata_2019-01.csv.gz)

**Step 1: Start the Database Containers**

```bash
# Navigate to your lab directory
cd databases-101-lab

# Start PostgreSQL and DuckDB containers in the background
docker compose up -d

# Verify containers are running (you should see postgresql_db and duckdb_cli containers)
docker ps
```

**Expected Output**: 
You should see two containers running with names like `postgresql_db` and `duckdb_cli`. If they're not running, check for error messages in the docker-compose output.

**Step 2: Connect to PostgreSQL**

```bash
# Connect to the PostgreSQL container and access the psql command line
docker exec -it postgresql_db psql -U postgres -d taxi_analysis
```

**Expected Output**: 

You should see the PostgreSQL prompt that looks like:

```
taxi_analysis=#
```

This means you're now connected to PostgreSQL and ready to run SQL commands. The `#` symbol indicates you have superuser privileges.

**Note for DuckDB**: 
When you need to connect to DuckDB later in the lab, you'll use the command `docker exec -it duckdb_cli /duckdb`. The `/duckdb` path is required because the DuckDB executable is located in the root directory of the container.

### Task 1: Data Import and Table Creation in PostgreSQL (8 minutes)

**Database: PostgreSQL**

**Step 1: Create the trips table structure**

```sql
-- Create the main trips table matching the actual NYC taxi dataset structure
-- Each column type is chosen to balance storage efficiency with data accuracy
CREATE TABLE trips (
    VendorID INTEGER,                     -- Taxi company identifier (1=Creative Mobile, 2=VeriFone Inc)
    tpep_pickup_datetime TIMESTAMP,      -- When the trip started (tpep = TLC Trip Record)
    tpep_dropoff_datetime TIMESTAMP,     -- When the trip ended
    passenger_count INTEGER,             -- Number of passengers (0-9, though some records may be 0)
    trip_distance DECIMAL(8,2),          -- Distance in miles (8 digits total, 2 after decimal)
    RatecodeID INTEGER,                  -- Rate code (1=Standard, 2=JFK, 3=Newark, 4=Nassau/Westchester, 5=Negotiated, 6=Group ride)
    store_and_fwd_flag CHAR(1),          -- Y/N flag indicating if trip was stored locally before sending to vendor
    PULocationID INTEGER,                -- Pickup location zone ID (references TLC taxi zones)
    DOLocationID INTEGER,                -- Dropoff location zone ID (references TLC taxi zones)  
    payment_type INTEGER,                -- Payment method (1=Credit card, 2=Cash, 3=No charge, 4=Dispute, 5=Unknown, 6=Voided trip)
    fare_amount DECIMAL(8,2),            -- Base fare amount calculated by the meter
    extra DECIMAL(8,2),                  -- Extra charges ($0.50 and $1 rush hour and overnight charges)
    mta_tax DECIMAL(8,2),                -- $0.50 MTA tax automatically triggered based on metered rate
    tip_amount DECIMAL(8,2),             -- Tip amount (automatically populated for credit card tips, cash tips not included)
    tolls_amount DECIMAL(8,2),           -- Total amount of all tolls paid in trip
    improvement_surcharge DECIMAL(8,2),  -- $0.30 improvement surcharge assessed on hailed trips
    total_amount DECIMAL(8,2),           -- Total amount charged to passengers (does not include cash tips)
    congestion_surcharge DECIMAL(8,2)    -- Congestion surcharge for trips in Manhattan south of 96th Street
);
```

**Expected Output**: 
You should see:
```
CREATE TABLE
```

This confirms that PostgreSQL has successfully created your table structure. The table is now ready to receive data but contains zero rows.

**Step 2: Import the CSV data**

```sql
-- Use PostgreSQL's COPY command for efficient bulk loading
-- This is much faster than individual INSERT statements
\COPY trips FROM '/data/taxi_trips.csv' DELIMITER ',' CSV HEADER;
```

**Expected Output**: You should see something like:
```
COPY 1000000
```

This means PostgreSQL successfully imported 1 million rows from the CSV file. The COPY command is PostgreSQL's optimized bulk loading mechanism, which can process thousands of rows per second.

**Step 3: Verify the import worked**

```sql
-- Check that data was imported correctly
SELECT COUNT(*) FROM trips;

-- Look at a sample of the imported data
SELECT * FROM trips LIMIT 5;
```

**Expected Output**: 
The count should show 1,000,000 rows, and the sample should display 5 actual taxi trip records with realistic data like timestamps, GPS coordinates, and fare amounts.

### Task 2: Index Impact Demonstration in PostgreSQL (10 minutes)

**Database: PostgreSQL**

**Step 1: Run a query without any indexes (baseline measurement)**

```sql
-- Enable timing to measure query performance
\timing on

-- Run an aggregation query that requires scanning date ranges
-- This query calculates hourly trip statistics for one week
SELECT DATE_TRUNC('hour', tpep_pickup_datetime) as hour,
       COUNT(*) as trip_count,
       AVG(fare_amount) as avg_fare
FROM trips 
WHERE tpep_pickup_datetime BETWEEN '2019-01-01' AND '2019-12-31'
GROUP BY DATE_TRUNC('hour', tpep_pickup_datetime)
ORDER BY hour;
```

**Expected Output**: You should see:
- Query results showing hourly aggregations (hour, trip count, average fare)
- A timing line like: `Time: 3847.234 ms (00:03.847)`

**Why This Is Slow**: Without an index on tpep_pickup_datetime, PostgreSQL must scan every single row in the table (called a "sequential scan") to find trips in the date range. It's reading approximately 8GB of data from disk.

**Step 2: Create a B-tree index on tpep_pickup_datetime**

```sql
-- Create an index that will dramatically speed up date range queries
CREATE INDEX idx_pickup_datetime ON trips(tpep_pickup_datetime);
```

**Expected Output**: You should see:

```
CREATE INDEX
```

The index creation might take 30-60 seconds because PostgreSQL is reading all 1 million rows, sorting them by pickup_datetime, and building a balanced tree structure.

**Step 3: Run the same query with the index**
```sql
-- Execute the identical query again - PostgreSQL will automatically use the index
SELECT DATE_TRUNC('hour', tpep_pickup_datetime) as hour,
       COUNT(*) as trip_count,
       AVG(fare_amount) as avg_fare
FROM trips 
WHERE tpep_pickup_datetime BETWEEN '2019-01-01' AND '2019-12-31'
GROUP BY DATE_TRUNC('hour', tpep_pickup_datetime)
ORDER BY hour;
```

**Expected Output**: You should see:
- The same query results as before
- A dramatically improved timing like: `Time: 245.123 ms`

**Why This Is Fast**: The index allows PostgreSQL to quickly locate only the rows within the date range without scanning the entire table. It's like jumping directly to the right section of a phone book instead of reading every page.

**Step 4: Examine the query execution plan**
```sql
-- Use EXPLAIN ANALYZE to see how PostgreSQL executes the query
-- This shows you the "behind the scenes" decision-making process
EXPLAIN (ANALYZE, BUFFERS) 
SELECT DATE_TRUNC('hour', tpep_pickup_datetime) as hour,
       COUNT(*) as trip_count,
       AVG(fare_amount) as avg_fare
FROM trips 
WHERE tpep_pickup_datetime BETWEEN '2019-01-01' AND '2019-12-31'
GROUP BY DATE_TRUNC('hour', tpep_pickup_datetime)
ORDER BY hour;
```

**Expected Output**: You'll see a detailed execution plan that includes:
- `Index Scan using idx_pickup_datetime` instead of `Seq Scan on trips`
- `Buffers: shared hit=X read=Y` showing how many pages were accessed
- Actual timing for each step of the query

**Key Learning Points**: The execution plan reveals that PostgreSQL now uses an "Index Scan" instead of a "Sequential Scan," and the number of buffer pages read is dramatically lower. This is concrete evidence of why indexes matter for performance.

### Task 3: Column Store Comparison with DuckDB (10 minutes)

**Database: DuckDB**

**Step 1: Connect to DuckDB**
```bash
# Open a new terminal and connect to the DuckDB container
docker exec -it duckdb_cli /duckdb
```

**Expected Output**: You should see the DuckDB prompt:
```
v1.1.3 dev
Enter ".help" for usage hints.
D 
```

The `D` prompt indicates you're now in DuckDB's command line interface.

**Step 2: Configure DuckDB for optimal CSV reading**
```sql
-- Set DuckDB to display headers for better readability
.mode csv
.headers on

-- Enable timing to compare with PostgreSQL
.timer on
```

**Expected Output**: DuckDB will show timing information after each query, similar to PostgreSQL's \timing on.

**Step 3: Query the CSV file directly (without importing)**
```sql
-- DuckDB can analyze CSV files directly without requiring an import step
-- This demonstrates how column-oriented databases optimize for analytical workloads
SELECT DATE_TRUNC('hour', tpep_pickup_datetime) as hour,
       COUNT(*) as trip_count,
       AVG(fare_amount) as avg_fare
FROM read_csv_auto('/data/taxi_trips.csv')
WHERE tpep_pickup_datetime BETWEEN '2019-01-01' AND '2019-12-31'
GROUP BY DATE_TRUNC('hour', tpep_pickup_datetime)
ORDER BY hour;
```

**Expected Output**: You should see:
- The same hourly aggregation results as PostgreSQL
- Timing that's likely faster than PostgreSQL, even without explicit indexes
- A timing line like: `Run Time: 0.89s`

**Why DuckDB Is Fast Here**: DuckDB is optimized for analytical queries and uses columnar storage internally. When it reads the CSV, it only loads the columns needed for your query (tpep_pickup_datetime and fare_amount), ignoring all other columns. It also applies advanced compression and vectorized processing.

**Step 4: Import data for additional experiments**
```sql
-- Create a permanent table in DuckDB for more complex analysis
CREATE TABLE trips AS 
SELECT * FROM read_csv_auto('/data/taxi_trips.csv');

-- Verify the import
SELECT COUNT(*) FROM trips;
```

**Expected Output**: DuckDB should confirm that 1,000,000 rows were imported, and this import will likely be faster than PostgreSQL's COPY command because of DuckDB's columnar optimizations.

### Task 4: WAL Performance Impact in PostgreSQL (8 minutes)

**Database: PostgreSQL**

**Step 1: Return to PostgreSQL and check current WAL settings**
```bash
# Return to your PostgreSQL terminal, or open a new connection
docker exec -it postgresql_db psql -U postgres -d taxi_analysis
```

```sql
-- Check the current synchronous_commit setting
SHOW synchronous_commit;
```

**Expected Output**: You should see:
```
 synchronous_commit 
--------------------
 on
(1 row)
```

This means PostgreSQL is configured for maximum durability - every transaction waits for the WAL to be written to disk before confirming success.

**Step 2: Create a test table for write performance measurement**
```sql
-- Create a simple table for insert benchmarking
CREATE TABLE insert_test (
    id SERIAL PRIMARY KEY,           -- Auto-incrementing integer
    timestamp TIMESTAMP DEFAULT NOW(), -- Automatically set to current time
    data TEXT                        -- Variable-length text field
);
```

**Step 3: Benchmark with synchronous_commit ON (maximum durability)**
```sql
-- Enable timing if not already on
\timing on

-- Perform a bulk insert operation with full durability guarantees
-- generate_series creates a sequence of numbers from 1 to 10,000
INSERT INTO insert_test (data) 
SELECT 'test data ' || generate_series(1, 10000);
```

**Expected Output**: You should see:
```
INSERT 0 10000
Time: 2847.234 ms (00:02.847)
```

**Why This Takes Time**: With synchronous_commit ON, PostgreSQL must write each transaction to the WAL file and ensure it's physically written to disk before confirming the insert. This guarantees that your data survives power failures but adds significant latency.

**Step 4: Disable synchronous_commit and test performance**
```sql
-- Temporarily disable synchronous_commit for this session
-- This trades some durability for much better write performance
SET synchronous_commit = OFF;

-- Clear the test table to start fresh
TRUNCATE insert_test;

-- Run the identical insert operation
INSERT INTO insert_test (data) 
SELECT 'test data ' || generate_series(1, 10000);
```

**Expected Output**: You should see:
```
INSERT 0 10000
Time: 487.123 ms
```

**Key Learning Point**: The insert should be dramatically faster (perhaps 5-10x) with synchronous_commit disabled. This demonstrates the trade-off between durability guarantees and performance. In production, you might use this setting for bulk loading operations where you can tolerate some data loss risk.

**Step 5: Reset to safe defaults**
```sql
-- Always reset to safe settings after experiments
SET synchronous_commit = ON;
```

### Task 5: Join Algorithm Exploration in PostgreSQL (4 minutes)

**Database: PostgreSQL**

**Step 1: Create a dimension table for zones**
```sql
-- Create a lookup table for NYC taxi zones
-- This simulates the actual TLC (Taxi & Limousine Commission) zone system
CREATE TABLE taxi_zones (
    LocationID INTEGER PRIMARY KEY,
    Borough VARCHAR(50),
    Zone VARCHAR(100),
    service_zone VARCHAR(20)  -- Yellow, Green, or Boro Zone
);

-- Insert sample zone data that matches actual NYC taxi zones
-- These are real zones from the official TLC dataset
INSERT INTO taxi_zones VALUES 
(1, 'EWR', 'Newark Airport', 'EWR'),
(2, 'Queens', 'Jamaica Bay', 'Boro Zone'),
(4, 'Manhattan', 'Alphabet City', 'Yellow Zone'),
(7, 'Queens', 'Astoria', 'Boro Zone'),
(13, 'Manhattan', 'Battery Park', 'Yellow Zone'),
(43, 'Manhattan', 'Central Park', 'Yellow Zone'),
(48, 'Manhattan', 'Clinton East', 'Yellow Zone'),
(79, 'Manhattan', 'East Village', 'Yellow Zone'),
(87, 'Manhattan', 'Financial District North', 'Yellow Zone'),
(88, 'Manhattan', 'Financial District South', 'Yellow Zone'),
(90, 'Manhattan', 'Flatiron', 'Yellow Zone'),
(100, 'Manhattan', 'Greenwich Village North', 'Yellow Zone'),
(107, 'Manhattan', 'Hamilton Heights', 'Boro Zone'),
(113, 'Manhattan', 'Hells Kitchen North', 'Yellow Zone'),
(114, 'Manhattan', 'Hells Kitchen South', 'Yellow Zone'),
(116, 'Manhattan', 'Highbridge Park', 'Boro Zone'),
(120, 'Manhattan', 'Inwood', 'Boro Zone'),
(125, 'Manhattan', 'Inwood Hill Park', 'Boro Zone'),
(127, 'Manhattan', 'Lenox Hill East', 'Yellow Zone'),
(128, 'Manhattan', 'Lenox Hill West', 'Yellow Zone'),
(132, 'Queens', 'JFK Airport', 'Airports'),
(138, 'Queens', 'LaGuardia Airport', 'Airports'),
(140, 'Manhattan', 'Lincoln Square East', 'Yellow Zone'),
(141, 'Manhattan', 'Lincoln Square West', 'Yellow Zone'),
(142, 'Manhattan', 'Little Italy/NoLiTa', 'Yellow Zone'),
(143, 'Manhattan', 'Lower East Side', 'Yellow Zone'),
(144, 'Manhattan', 'Meatpacking District', 'Yellow Zone'),
(148, 'Manhattan', 'Midtown Center', 'Yellow Zone'),
(151, 'Manhattan', 'Midtown North', 'Yellow Zone'),
(152, 'Manhattan', 'Midtown South', 'Yellow Zone'),
(158, 'Manhattan', 'Murray Hill', 'Yellow Zone'),
(161, 'Manhattan', 'Midtown East', 'Yellow Zone'),
(162, 'Manhattan', 'Penn Station/Madison Sq West', 'Yellow Zone'),
(163, 'Manhattan', 'Port Authority Bus Terminal', 'Yellow Zone'),
(164, 'Manhattan', 'Randalls Island', 'Yellow Zone'),
(166, 'Manhattan', 'Roosevelt Island', 'Yellow Zone'),
(170, 'Manhattan', 'Seaport', 'Yellow Zone'),
(186, 'Manhattan', 'Times Square/Theatre District', 'Yellow Zone'),
(202, 'Manhattan', 'Tomkins Square Park', 'Yellow Zone'),
(209, 'Manhattan', 'Tribeca/Civic Center', 'Yellow Zone'),
(211, 'Manhattan', 'Two Bridges/Seward Park', 'Yellow Zone'),
(224, 'Manhattan', 'Upper East Side North', 'Yellow Zone'),
(229, 'Manhattan', 'Upper East Side South', 'Yellow Zone'),
(230, 'Manhattan', 'Upper West Side North', 'Yellow Zone'),
(231, 'Manhattan', 'Upper West Side South', 'Yellow Zone'),
(232, 'Manhattan', 'Waldorf Astoria/General Electric Building', 'Yellow Zone'),
(233, 'Manhattan', 'Washington Heights North', 'Boro Zone'),
(234, 'Manhattan', 'Washington Heights South', 'Boro Zone'),
(236, 'Manhattan', 'West Chelsea/Hudson Yards', 'Yellow Zone'),
(237, 'Manhattan', 'West Village', 'Yellow Zone'),
(238, 'Manhattan', 'Westside', 'Yellow Zone'),
(239, 'Manhattan', 'Yorkville East', 'Yellow Zone'),
(243, 'Manhattan', 'World Trade Center', 'Yellow Zone'),
(244, 'Manhattan', 'Yankee Stadium', 'Boro Zone'),
(249, 'Manhattan', 'Garment District', 'Yellow Zone'),
(261, 'Manhattan', 'World Trade Center', 'Yellow Zone'),
(262, 'Manhattan', 'Yorkville West', 'Yellow Zone'),
(263, 'Manhattan', 'Sutton Place/Turtle Bay North', 'Yellow Zone');
```

**Step 2: Understanding the location ID system**
```sql
-- The trips table already contains PULocationID (pickup) and DOLocationID (dropoff)
-- These reference the LocationID in our taxi_zones table
-- Let's examine how these IDs are distributed in our dataset
SELECT PULocationID, COUNT(*) as pickup_count
FROM trips 
WHERE tpep_pickup_datetime BETWEEN '2019-01-01' AND '2023-01-02'
GROUP BY PULocationID 
ORDER BY pickup_count DESC 
LIMIT 10;
```

**Expected Output**: You should see the most popular pickup locations by their ID numbers, with Manhattan locations likely dominating the results.

**Step 3: Run a join query without indexes and examine the plan**
```sql
-- Join trips with zones to get human-readable zone names
-- EXPLAIN ANALYZE shows us which join algorithm PostgreSQL chooses
EXPLAIN (ANALYZE, BUFFERS)
SELECT z.Zone, 
       z.Borough,
       COUNT(*) as trip_count,
       AVG(t.fare_amount) as avg_fare
FROM trips t
JOIN taxi_zones z ON t.PULocationID = z.LocationID
WHERE t.tpep_pickup_datetime BETWEEN '2019-01-01' AND '2023-01-02'
GROUP BY z.Zone, z.Borough
ORDER BY trip_count DESC
LIMIT 20;
```

**Expected Output**: Look for these key elements in the execution plan:
- Join algorithm: Likely "Hash Join" because taxi_zones is small
- Node type on trips table: "Seq Scan" or "Index Scan" depending on whether it uses our datetime index
- Buffer statistics showing how many pages were read

**Step 4: Create an index on the join column and observe the change**
```sql
-- Create an index on the PULocationID column
CREATE INDEX idx_pickup_location ON trips(PULocationID);

-- Run the same query and compare the execution plan
EXPLAIN (ANALYZE, BUFFERS)
SELECT z.Zone, 
       z.Borough,
       COUNT(*) as trip_count,
       AVG(t.fare_amount) as avg_fare
FROM trips t
JOIN taxi_zones z ON t.PULocationID = z.LocationID
WHERE t.tpep_pickup_datetime BETWEEN '2019-01-01' AND '2023-01-02'
GROUP BY z.Zone, z.Borough
ORDER BY trip_count DESC
LIMIT 20;
```

**Expected Output**: The execution plan might change to show:
- Different join algorithm (possibly "Nested Loop" for small result sets)
- "Index Scan" instead of "Seq Scan" on the trips table
- Lower buffer read counts
- Faster overall execution time

**Key Learning Point**: PostgreSQL's query planner dynamically chooses join algorithms based on table sizes, available indexes, and estimated result set sizes. Understanding this helps you create indexes that support your most important query patterns.

### Task 6: Cleanup and Reflection (5 minutes)

**Database: Both PostgreSQL and DuckDB**

**Step 1: Review what we've learned**
```sql
-- In PostgreSQL, check our created objects
\dt  -- List all tables
\di  -- List all indexes
```

**Step 2: Clean up test data (optional)**
```sql
-- Remove test tables if desired
DROP TABLE IF EXISTS insert_test;
-- Keep trips and dim_zones for any additional experimentation
```

**Step 3: Exit database connections**
```sql
-- In PostgreSQL
\q

-- In DuckDB
.quit
```

**Step 4: Stop containers**
```bash
# Back in your terminal, stop the containers
docker-compose down

# view and delete the container volumes that were created
docker volume ls
docker volume rm <volume name>
```

---

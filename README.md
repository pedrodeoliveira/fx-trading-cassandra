# FX Trading using a Cassandra Database

This project provides a list of the *access patterns* (queries) for a FX Trading algo. 
The definition of access patterns allows one to build a **Logical Data Model** for the 
Cassandra database supporting this trading algo.

Some queries cannot be made efficiently using CQL, due to Cassandra's design. Queries that 
access multiple partitions or with complex aggregations require
additional technologies. As such, **Spark** will be used on top of the Cassandra
database in order to efficiently process this type of operations. The queries
that require the use of Spark will be properly identified.

Some tables were previously defined as a *materialized view*, but these were not
working properly, as such, they are currently being update using **Spark**. 
The **Status** column will identify these columns.

**Query ID**|**Component**|**Table**|**Spark Table**|**Spark Columns**|**Status**
:-----|:-----|:-----:|:-----:|:-----:
Trading|`trading_status`|Y|N|OK
Trading|`trading_results`|Y|N|**Not Done**
Trading|`daily_status`|Y|N|OK
Trading|`daily_results`|Y|N|OK
Trading|`daily_returns`|Y|N|**Not Done**
Trading|`daily_transactions_by_type`|Y|N|OK
Top|`top_brokers_order_unrealized`|Y|N|OK
Top|`top_accounts_order_unrealized`|Y|N|OK
Top|`top_instances_order_unrealized`|Y|N|OK
Top|`top_positions_order_unrealized`|Y|N|OK
Top|`top_trades_order_unrealized`|Y|N|OK
Strategy|`strategy_status`|Y|N|OK
Strategy|`transactions_by_strategy_type`|Y|N|Should be a MV
Strategy|`daily_transactions_by_strategy_type`|Y|N|OK
Strategy|`monthly_returns_by_strategy`|Y|N|**Not done**
Strategy|`daily_status_by_strategy`|Y|N|STARTED
Strategy|`daily_results_by_strategy`|Y|N|OK
Strategy|`daily_returns_by_strategy`|Y|N|**Not done**
Strategy|`monthly_returns_by_strategy`|Y|N|**Not done**
Brokers|`broker_status`|Y|N|OK
Brokers|`daily_results_by_broker`|Y|N|OK
Brokers|`daily_status_by_broker`|Y|N|OK
Brokers|`instrument_required_margins_by_broker`|N|N|**Migrate**
Accounts|`account`|N|N|OK
Accounts|`account_status`|N|Y|OK
Accounts|`account_status_by_id_day`|N|N|OK
Accounts|`account_returns_by_strategy`|Y|N|**Not done**
Accounts|`daily_results_by_account`|Y|N|OK
Accounts|`daily_status_by_account`|Y|N|OK
Accounts|`transactions_by_account_type`|N|N|OK
Instances|`instance_status_by_id_day`|N|N|OK
Instances|`instance_status_by_strategy`|N|Y|OK
Instances|`instance_status_by_account`|Y|N|Should be a MV
Instances|`instance_returns_by_strategy`|Y|N|**Not Done**
Instances|`transactions_by_instance_type`|Y|N|OK
Instances|`logs_by_instance`|N|N|OK
Instances|`daily_results_by_instance`|Y|N|OK
Instances|`hourly_status_by_instance`|N|N|OK
Instances|`daily_status_by_instance`|Y|N|OK
Instances|`daily_transactions_by_instance_type`|Y|N|OK
Risk Analysis|`account_margins`|Y|N|OK
Instruments|`instruments`|N|N|OK
Instruments|`symbol_bias_by_day`|N|N|OK
Instruments|`symbol_bias_last_update`|N|N|OK
Instruments|`instruments_summary_by_strategy`|Y|N|OK
Instruments|`oanda_prices_by_symbol`|N|N|OK
Closed Trades|`closed_trades_by_instance`|N|N|OK
Closed Trades|`closed_trades_by_account`|Y|N|Should be a MV
Closed Trades|`closed_trades_by_strategy`|Y|N|Should be a MV
Closed Trades|`virtual_stops_by_strategy`|Y|N|Should be MV
Closed Trades|`last_week_closed_trades_by_instance`|Y|N|OK
Closed Positions|`closed_positions_by_instance`|Y|N|OK
Closed Positions|`closed_positions_by_strategy`|Y|N|Should be a MV
Closed Positions|`last_week_closed_positions_by_instance`|Y|N|OK
Open Trades|`open_trades_by_instance`|N|Y|**Incomplete**
Open Trades|`open_trades_by_strategy`|Y|N|Should be a MV
Open Trades|`open_trades_by_account`|Y|N|Should be a MV
Open Positions|`open_positions_by_instance`|Y|N|OK
Open Positions|`open_positions_by_strategy`|Y|N|Should be a MV
Live Environment|`critical_variables_by_instance_id_instrument`|N|N|OK

Major points to review/analyze:

- How the `dummy` columns should be filled.
- Frequency of batch jobs and tables that write with specified **TTL**.
- Consistency level when reading/writing using Spark, default is `LOCAL_ONE`. The name of the property for the *output* configuration is `spark.cassandra.output.consistency.level`.
- Missing inclusion of **OANDA** swaps into account, broker and strategy aggregations.
- Review roundings on profits.
- The `expected_profit` is missing in the terminal, broker and strategy levels, but is already being calculated in the `open_positions_by_strategy`. Check if aggregation should be done in the `account_status_by_strategy` table or in another table.

Notes:

- Some columns that were initially defined as *static* are not anymore, because they are used in *materialized views* that forbid the use of such columns.

## 1. Trading

### trading_status

- Obtain the trading status or summary of results of all the strategies.
- This table is updated by a Spark batch job, that aggregates the information of the `strategy_status` table.
- In this case the `dummy` column is required given that we cannot have tables without a **primary key** (a primary key identifies the location and order of data storage).

```cql
CREATE TABLE trading_status (dummy INT,
    net_investments DOUBLE,
    balance DOUBLE,
    equity DOUBLE,
    unrealized_profit DOUBLE,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    net_profit DOUBLE,
    most_delayed TIMESTAMP,
    margin_used DOUBLE,
    margin_used_pct DOUBLE,
    margin_free DOUBLE,
    highest_mu_pct DOUBLE,
    total_size DOUBLE,
    max_size DOUBLE,
    cnt_open_pos INT,
    cnt_open_trades INT,
    nbr_instances INT,
    nbr_brokers INT,
    nbr_accounts INT,
    nbr_strategies INT,
    net_open_positions_eur DOUBLE,
    gross_open_positions_eur DOUBLE,
    net_open_positions_usd DOUBLE,
    gross_open_positions_usd DOUBLE,
    exposures_map MAP<TEXT, DOUBLE>,
    PRIMARY KEY (dummy));
```

### trading_results

- Obtain the aggregate of the trading results for all strategies.
- This table is updated by a Spark batch job from tables `daily_results` and `daily_returns` with `date == today()`.
- All columns with exception of `itd_return` and `annualized_return` reference information of the most recent trading day.

```cql
CREATE TABLE trading_results (dummy INT,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    nbr_closed_positions INT,
    nbr_closed_trades INT,
    max_size INT,
    avg_profit DOUBLE,
    max_profit DOUBLE,
    return DOUBLE,
    itd_return DOUBLE,
    annualized_return DOUBLE,
    PRIMARY KEY (dummy));
```

### daily_status

- Find the *status** grouped by `date` for all strategies/instances and between an interval of **days**. Order by `date` (day) descending.
- The `date` has the format `YYYY-MM-DD`.
- This table is updated by a Spark batch job, that aggregates the daily account's status by strategy of the table **daily_instance_status_by_id**.

```cql
CREATE TABLE daily_status (dummy INT,
    date TEXT,
    last_update TIMESTAMP,
    net_investments DOUBLE,
    balance DOUBLE,
    equity DOUBLE,
    unrealized_profit DOUBLE,
    margin_used DOUBLE,
    margin_used_pct DOUBLE,
    margin_free DOUBLE,
    total_size DOUBLE,
    max_size DOUBLE,
    cnt_open_pos INT,
    cnt_open_trades INT,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    net_profit DOUBLE,
    nbr_instances INT,
    nbr_brokers INT,
    nbr_accounts INT,
    nbr_strategies INT,
    PRIMARY KEY (dummy, date))
WITH CLUSTERING ORDER BY (date ASC);
```

### daily_results

- Obtain the daily results of all aggregated strategies.
- This table is updated by a Spark job that aggregates the information on the table `daily_results_by_strategy`.
- We need to review when making a query without specifying the `dummy` partition key if the results are ordered by `date` or not.

```cql
CREATE TABLE daily_results (dummy INT,
    date TEXT,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    nbr_closed_positions INT,
    nbr_closed_trades INT,
    max_size INT,
    avg_profit DOUBLE,
    max_profit DOUBLE,
    PRIMARY KEY (dummy, date))
WITH CLUSTERING ORDER BY (date ASC);
```

### daily_returns

- Obtain the daily returns of all aggregated strategies.
- This table is updated by a Spark job that

```cql
CREATE TABLE daily_returns (dummy INT,
    date TEXT,
    equity DOUBLE,
    net_investments DOUBLE,
    return DOUBLE,
    nav DOUBLE,
    itd_return DOUBLE,
    annualized_return DOUBLE,
    PRIMARY KEY (dummy, date))
WITH CLUSTERING ORDER BY (date ASC);
```

### daily_transactions_by_type

- Find the **transactions** grouped by `date` for a given transaction `type` and between an interval of **days**. Order by `date` descending.
- The `date` has the format `YYYY-MM-DD`.
- This table is updated by a Spark batch job, that aggregates the transactions by day of the table **transactions_by_strategy_type**.

```cql
CREATE TABLE daily_transactions_by_type (type TEXT,
    date TEXT,
    amount DOUBLE,
    PRIMARY KEY (type, date))
WITH CLUSTERING ORDER BY (date DESC);
```

## 2. Top

The top referes to the first/last **N** records.

### top_brokers_order_unrealized

- Finds the list of 20 top **brokers** with highest and lowest `unrealized_profit`. By default results are ordered by `broker_id` ascending and sorting by `unrealized_profit` will have to be done at the client level (Cassandra does not allow ordering by non-clustering columns).
- This table is updated by a Spark batch job, which obtains the positions info from the table **brokers**.

```cql
CREATE TABLE top_brokers_order_unrealized (broker_id TEXT,
    nbr_accounts INT,
    most_delayed TIMESTAMP,
    net_investments DOUBLE,
    balance DOUBLE,
    equity DOUBLE,
    unrealized_profit DOUBLE,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    net_profit DOUBLE,
    margin_used DOUBLE,
    margin_used_pct DOUBLE,
    margin_free DOUBLE,
    total_size DOUBLE,
    max_size DOUBLE,
    cnt_open_trades INT,
    cnt_open_pos INT,
    PRIMARY KEY (broker_id));
```

### top_accounts_order_unrealized

- Find the list of top **accounts**, with positive `balance`, with the highest and lowest `unrealized_profit`, by `account_id`.
- This table is updated by a Spark batch job, which obtains the positions info from the table `account_status`.

```cql
CREATE TABLE top_accounts_order_unrealized (account_id TEXT,
    broker_id TEXT,
    last_update TIMESTAMP,
    net_investments DOUBLE,
    balance DOUBLE,
    equity DOUBLE,
    unrealized_profit DOUBLE,
    margin_used DOUBLE,
    margin_used_pct DOUBLE,
    margin_free DOUBLE,
    total_size DOUBLE,
    max_size DOUBLE,
    cnt_open_pos INT,
    cnt_open_trades INT,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    net_profit DOUBLE,
    net_open_positions_eur DOUBLE,
    net_open_positions_usd DOUBLE,
    gross_open_positions_eur DOUBLE,
    gross_open_positions_usd DOUBLE,
    PRIMARY KEY (account_id));
```

### top_instances_order_unrealized

- Find the list of top **instances**, with positive `balance`, with the highest and lowest `unrealized_profit` and order results by `instance_id`.
- This table is updated by a Spark batch job, which obtains the positions info from the table **instance_status_by_strategy**.

```cql
CREATE TABLE top_instances_order_unrealized (instance_id TEXT,
    strategy_id TEXT,
    account_id TEXT,
    broker_id TEXT,
    last_update TIMESTAMP,
    net_investments DOUBLE,
    balance DOUBLE,
    equity DOUBLE,
    unrealized_profit DOUBLE,
    margin_used DOUBLE,
    margin_used_pct DOUBLE,
    margin_free DOUBLE,
    total_size DOUBLE,
    max_size DOUBLE,
    cnt_open_pos INT,
    cnt_open_trades INT,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    net_profit DOUBLE,
    PRIMARY KEY (instance_id));
```

### top_positions_order_unrealized

- Find the list of top 20 **positions** the highest and lowest `unrealized_profit`.
- This table is updated by a Spark batch job, which obtains the positions info from the table **open_positions_by_instance**.

```cql
CREATE TABLE top_positions_order_unrealized (position_id TEXT,
    strategy_id TEXT,
    instance_id TEXT,
    account_id TEXT,
    broker_id TEXT,
    symbol TEXT,
    direction TEXT,
    max_size DOUBLE,
    size DOUBLE,
    nbr_trades DOUBLE,
    open_time TIMESTAMP,
    duration DOUBLE,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    initial_size DOUBLE,
    expected_profit DOUBLE,
    units_base_ccy DOUBLE,
    units_quote_ccy DOUBLE,
    margin_used DOUBLE,
    margin_next DOUBLE,
    margin_used_next DOUBLE,
    weight_pct DOUBLE,
    PRIMARY KEY (position_id));
```

### top_trades_order_unrealized

- Find the list of top 20 **trades** the highest and lowest `unrealized_profit`.
- This table is updated by a Spark batch job, which obtains the positions info from the table **open_trades_by_instance**.

```cql
CREATE TABLE top_trades_order_unrealized (trade_id TEXT,
    strategy_id TEXT,
    instance_id TEXT,
    account_id TEXT,
    broker_id TEXT,
    position_id TEXT,
    symbol TEXT,
    direction TEXT,
    size DOUBLE,
    open_time TIMESTAMP,
    duration DOUBLE,
    open_price DOUBLE,
    last_price DOUBLE,
    pips DOUBLE,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    units_base_ccy DOUBLE,
    units_quote_ccy DOUBLE,
    margin_required DOUBLE,
    margin_used DOUBLE,
    weight_pct DOUBLE,
    PRIMARY KEY (instance_id, trade_id))
WITH CLUSTERING ORDER BY (trade_id ASC);
```

## 3. Strategy

### strategy_status

- Obtain the strategies' status or summary of results by `strategy_id`.
- This table is updated by a Spark batch job, that aggregates the information of the `instance_status_by_strategy` table.

```cql
CREATE TABLE strategy_status (strategy_id TEXT,
    net_investments DOUBLE,
    balance DOUBLE,
    equity DOUBLE,
    unrealized_profit DOUBLE,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    net_profit DOUBLE,
    most_delayed TIMESTAMP,
    margin_used DOUBLE,
    margin_used_pct DOUBLE,
    margin_free DOUBLE,
    highest_mu_pct DOUBLE,
    total_size DOUBLE,
    max_size DOUBLE,
    cnt_open_pos INT,
    cnt_open_trades INT,
    nbr_instances INT,
    nbr_brokers INT,
    nbr_accounts INT,
    net_open_positions_eur DOUBLE,
    gross_open_positions_eur DOUBLE,
    net_open_positions_usd DOUBLE,
    gross_open_positions_usd DOUBLE,
    exposures_map MAP<TEXT, DOUBLE>,
    PRIMARY KEY (strategy_id));
```

### transactions_by_strategy_type

- Find all the **transactions** by `strategy_id`, transaction `type` and between an interval of **days**. Order by `tstamp` descending.
- The `date` has the format `YYYY-MM-DD`.
- This table is update by a Spark batch job based on the information of the tables `instance_status_by_strategy` and `transactions_by_account_type`.

```cql
CREATE TABLE transactions_by_strategy_type (type TEXT,
    strategy_id TEXT,
    account_id TEXT,
    broker_id TEXT,
    instance_id TEXT,
    date TEXT,
    tstamp TIMESTAMP,
    amount DOUBLE,
    transaction_id TEXT,
    comment TEXT,
    PRIMARY KEY ((strategy_id, type), date, tstamp, account_id))
WITH CLUSTERING ORDER BY (date DESC, tstamp DESC, account_id ASC);
```

Previous definition as a *materialized view*:

```cql
CREATE MATERIALIZED VIEW transactions_by_strategy_type as
SELECT *
FROM transactions_by_instance_type
WHERE instance_id IS NOT NULL and type IS NOT NULL and tstamp is NOT NULL
PRIMARY KEY(strategy_id, type, instance_id, tstamp);
```

### daily_transactions_by_strategy_type

- Find the **transactions** grouped by `day` for a given `strategy_id`, transaction `type` and between an interval of **days**. Order by `day` descending.
- The `date` has the format `YYYY-MM-DD`.
- This table is updated by a Spark batch job, that aggregates the transactions by day of the table **transactions_by_strategy_type**.

```cql
CREATE TABLE daily_transactions_by_strategy_type (strategy_id TEXT,
    type TEXT,
    date TEXT,
    amount DOUBLE,
    PRIMARY KEY ((strategy_id, type), date))
WITH CLUSTERING ORDER BY (date DESC);
```

### daily_status_by_strategy

- Find the *status** grouped by `day` for a given `strategy_id` and between an interval of **days**. Order by `day` descending.
- The `date` has the format `YYYY-MM-DD`.
- This table is updated by a Spark batch job, that aggregates the daily account's status by strategy of the table **daily_instance_status_by_id**.

```cql
CREATE TABLE daily_status_by_strategy (strategy_id TEXT,
    date TEXT,
    last_update TIMESTAMP,
    net_investments DOUBLE,
    balance DOUBLE,
    equity DOUBLE,
    unrealized_profit DOUBLE,
    margin_used DOUBLE,
    margin_used_pct DOUBLE,
    margin_free DOUBLE,
    total_size DOUBLE,
    max_size DOUBLE,
    cnt_open_pos INT,
    cnt_open_trades INT,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    net_profit DOUBLE,
    nbr_instances INT,
    nbr_brokers INT,
    nbr_accounts INT,
    PRIMARY KEY (strategy_id, date))
WITH CLUSTERING ORDER BY (date ASC);
```

### daily_results_by_strategy

- Find the strategy results (profit, number of closed positions, ...) by `strategy_id` and grouped by `date` (day). Results ordered by `date` ascending.
- This table is updated by a Spark batch job that aggregates the information from the table `daily_results_by_instance`.
- The `date` has the format `YYYY-MM-DD`.

```cql
CREATE TABLE daily_results_by_strategy (strategy_id TEXT,
    date TEXT,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    nbr_closed_positions INT,
    nbr_closed_trades INT,
    max_size INT,
    avg_profit DOUBLE,
    max_profit DOUBLE,
    PRIMARY KEY (strategy_id, date))
WITH CLUSTERING ORDER BY (date ASC);
```

### daily_returns_by_strategy

- Find the daily returns by `strategy_id`. Results ordered by `date` ascending.
- This table is updated by a Spark batch job that aggregates the information from the table **closed_positions_by_instance**, **transactions_by_strategy_type** and  **status_by_strategy**.
- The `date` has the format `YYYY-MM-DD`.
- Added `closed_return`, `equity`, `return` and `nav` column that should be updated my spark job.

```cql
CREATE TABLE daily_returns_by_strategy (strategy_id TEXT,
    date TEXT,
    equity DOUBLE,
    net_investments DOUBLE,
    return DOUBLE,
    nav DOUBLE,
    itd_return DOUBLE,
    annualized_return DOUBLE,
    PRIMARY KEY (strategy_id, date))
WITH CLUSTERING ORDER BY (date ASC);
```

### monthly_returns_by_strategy

- Find the strategy returns by `strategy_id` and grouped by the month of `date` (day). Results ordered by `date` ascending.
- This table is updated by a Spark batch job that aggregates the information from the table `daily_returns_by_strategy`.
- The `date` has the format `YYYY-MM-DD`.

```cql
CREATE TABLE monthly_returns_by_strategy (strategy_id TEXT,
    date TEXT,
    return DOUBLE,
    nav DOUBLE,
    equity DOUBLE,
    closed_return DOUBLE,
    return_cum DOUBLE,
    PRIMARY KEY (strategy_id, date))
WITH CLUSTERING ORDER BY (date ASC);
```

## 4. Brokers

### broker_status

- Obtain the brokers' status and order by `broker_id` ascending.
- This table is updated by a Spark batch job, which aggregates the information from the `account_status` table.
- The `highest_mu_pct` is referred to the highest margin used in percentage of all accounts associated with this `broker_id`.

```cql
CREATE TABLE broker_status (broker_id TEXT,
    nbr_accounts INT,
    nbr_instances INT,
    most_delayed TIMESTAMP,
    net_investments DOUBLE,
    balance DOUBLE,
    equity DOUBLE,
    unrealized_profit DOUBLE,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    net_profit DOUBLE,
    margin_used DOUBLE,
    margin_used_pct DOUBLE,
    highest_mu_pct DOUBLE,
    margin_free DOUBLE,
    total_size DOUBLE,
    max_size DOUBLE,
    cnt_open_trades INT,
    cnt_open_pos INT,
    net_open_positions_eur DOUBLE,
    gross_open_positions_eur DOUBLE,
    net_open_positions_usd DOUBLE,
    gross_open_positions_usd DOUBLE,
    exposures_map MAP<TEXT, DOUBLE>,
    PRIMARY KEY (broker_id));
```

### daily_results_by_broker

- Find the broker results (profit, number of closed positions, ...) by `broker_id` and grouped by `date` (day). Results ordered by `date` ascending.
- This table is updated by a Spark batch job that aggregates the information from the table `daily_results_by_instance`.
- The `date` has the format `YYYY-MM-DD`.

```cql
CREATE TABLE daily_results_by_broker (broker_id TEXT,
    date TEXT,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    nbr_closed_positions INT,
    nbr_closed_trades INT,
    max_size INT,
    avg_profit DOUBLE,
    max_profit DOUBLE,
    PRIMARY KEY (broker_id, date))
WITH CLUSTERING ORDER BY (date ASC);
```

### instrument_required_margins_by_broker

- Find the `margin_required` by `broker_id` and `symbol`.
- This table has been migrated from **MySQL**.

```cql
CREATE TABLE instrument_required_margins_by_broker (broker_id TEXT,
    symbol TEXT,
    margin_required DOUBLE,
    PRIMARY KEY (broker_id, symbol))
WITH CLUSTERING ORDER BY (symbol ASC);
```

### daily_status_by_broker

- Find the broker status grouped by `date` (day) for a given `broker_id` and between an interval of **days**. Order by `date` descending.
- The `date` has the format `YYYY-MM-DD`.
- This table is updated by a Spark batch job, that aggregates the daily instance status by `broker_id` of the table `daily_status_by_instance`.

```cql
CREATE TABLE daily_status_by_broker (broker_id TEXT,
    date TEXT,
    last_update TIMESTAMP,
    net_investments DOUBLE,
    balance DOUBLE,
    equity DOUBLE,
    unrealized_profit DOUBLE,
    margin_used DOUBLE,
    margin_used_pct DOUBLE,
    margin_free DOUBLE,
    total_size DOUBLE,
    max_size DOUBLE,
    cnt_open_pos INT,
    cnt_open_trades INT,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    net_profit DOUBLE,
    nbr_accounts INT,
    nbr_instances INT,
    PRIMARY KEY (broker_id, date))
WITH CLUSTERING ORDER BY (date ASC);
```

## 5. Accounts

### accounts

- Find all the accounts.

```cql
CREATE TABLE accounts (account_id TEXT,
    broker_id TEXT,
    base_ccy TEXT,
    trade_mode TEXT,
    hedge_allowed BOOLEAN,
    hedge_mode TEXT,
    leverage INT,
    inception_date TEXT,
    limit_orders INT,
    margin DOUBLE,
    margin_call DOUBLE,
    stop_out DOUBLE,
    reported TEXT,
    notes TEXT,
    PRIMARY KEY (account_id));
```

### account_status

- Find the last (most recent) account status for all accounts.
- Results ordered by `broker_id` and `account_id` ascending.
- The following columns will be updated by a Spark job: `net_investments`, `commission`, `swap`,`gross_profit`, `profit`, `net_profit`, `net_open_positions_eur/usd`, `gross_open_positions/usd`, `nbr_instances` and `exposures_map`.

```cql
CREATE TABLE account_status (account_id TEXT,
    broker_id TEXT,
    last_update TIMESTAMP,
    net_investments DOUBLE,
    balance DOUBLE,
    equity DOUBLE,
    unrealized_profit DOUBLE,
    margin_used DOUBLE,
    margin_used_pct DOUBLE,
    margin_free DOUBLE,
    total_size DOUBLE,
    max_size DOUBLE,
    cnt_open_pos INT,
    cnt_open_trades INT,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    net_profit DOUBLE,
    nbr_instances INT,
    net_open_positions_eur DOUBLE,
    gross_open_positions_eur DOUBLE,
    net_open_positions_usd DOUBLE,
    gross_open_positions_usd DOUBLE,
    exposures_map MAP<TEXT, DOUBLE>,
    PRIMARY KEY (account_id));
```

### account_status_by_id_day

- Find all account status as given directly by the broker, by `account_id` and  `date`, ordered by `tstamp_gmt` ascending.
- The `date` has the format `YYYY-MM-DD`.

```cql
CREATE TABLE account_status_by_id_day (strategy_id TEXT,
    date TEXT,
    account_id TEXT,
    broker_id TEXT static,
    tstamp TIMESTAMP,
    balance DOUBLE,
    equity DOUBLE,
    unrealized_profit DOUBLE,
    margin_used DOUBLE,
    margin_free DOUBLE,
    margin_used_pct DOUBLE,
    total_size DOUBLE,
    max_size DOUBLE,
    cnt_open_pos INT,
    cnt_open_trades INT,
    PRIMARY KEY ((account_id, date), tstamp))
WITH CLUSTERING ORDER BY (tstamp ASC);
```

### daily_results_by_account

- Find the daily results by `account_id` and `date` (day).
- The `date` has the format `YYYY-MM-DD`.
- This table is created by a Spark job from table `daily_results_by_instance`.

```cql
CREATE TABLE daily_results_by_account (account_id TEXT,
    broker_id TEXT static,
    date TEXT,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    nbr_closed_positions INT,
    nbr_closed_trades INT,
    max_size INT,
    avg_profit DOUBLE,
    max_profit DOUBLE,
    PRIMARY KEY (account_id, date))
WITH CLUSTERING ORDER BY (date ASC);
```

### daily_status_by_account

- Find the daily account status by `account_id` and ordered by `date` (day) ascending.
- The `date` has the format `YYYY-MM-DD`.
- This table is created by a Spark job from table `daily_status_by_instance`.

```cql
CREATE TABLE daily_status_by_account (account_id TEXT,
    date TEXT,
    broker_id TEXT static,
    last_update TIMESTAMP,
    net_investments DOUBLE,
    balance DOUBLE,
    equity DOUBLE,
    unrealized_profit DOUBLE,
    margin_used DOUBLE,
    margin_used_pct DOUBLE,
    margin_free DOUBLE,
    total_size DOUBLE,
    max_size DOUBLE,
    cnt_open_pos INT,
    cnt_open_trades INT,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    net_profit DOUBLE,
    nbr_instances INT,
    PRIMARY KEY (account_id, date))
WITH CLUSTERING ORDER BY (date ASC);
```

### account_returns_by_strategy

- Find the account return results by `strategy_id` and order results by `account_id` ascending.
- Spark table created using tables `account_status_by_strategy`, `closed_positions_by_strategy` and `transactions_by_account_type`

```cql
CREATE TABLE account_returns_by_strategy (strategy_id TEXT,
    account_id TEXT,
    inception_date TEXT,
    investments DOUBLE,
    withdrawals DOUBLE,
    net_investments DOUBLE,
    equity DOUBLE,
    ann_return DOUBLE,
    return_pct DOUBLE,
    PRIMARY KEY (strategy_id, account_id))
WITH CLUSTERING ORDER BY (account_id ASC);
```

### transactions_by_account_type

- Find all the **transactions** by `account_id`, transaction `type` and between an interval of **days**. Order by `tstamp_gmt` descending.
- The `date` has the format `YYYY-MM-DD`.

```cql
CREATE TABLE transactions_by_account_type (type TEXT,
    account_id TEXT,
    broker_id TEXT static,
    date TEXT,
    tstamp TIMESTAMP,
    amount DOUBLE,
    transaction_id TEXT,
    comment TEXT,
    PRIMARY KEY ((account_id, type), date, tstamp))
WITH CLUSTERING ORDER BY (date DESC, tstamp DESC);
```

## 6. Instances

### instance_status_by_id_day

- Find all **instance status** by `instance_id` and `date`, ordered by `tstamp` ascending.
- The `date` has the format `YYYY-MM-DD`.

```cql
CREATE TABLE instance_status_by_id_day (instance_id TEXT,
    date TEXT,
    strategy_id TEXT static,
    broker_id TEXT static,
    account_id TEXT static,
    tstamp TIMESTAMP,
    balance DOUBLE,
    equity DOUBLE,
    unrealized_profit DOUBLE,
    margin_used DOUBLE,
    margin_free DOUBLE,
    margin_used_pct DOUBLE,
    total_size DOUBLE,
    max_size DOUBLE,
    cnt_open_pos INT,
    cnt_open_trades INT,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    net_profit DOUBLE,
    PRIMARY KEY ((instance_id, date), tstamp))
WITH CLUSTERING ORDER BY (tstamp ASC);
```

### instance_status_by_strategy

- Find the last (most recent) `instance status` per `instance_id` for a given `strategy_id`. Results ordered by `instance_id` ascending.
- The following columns will be updated by a Spark job: `net_investments` `net_open_positions_eur/usd`, `gross_open_positions_eur/usd` and `exposures_map`.
- The job queries `closed_positions_by_strategy` for profit data and `transactions_by_strategy_type` for information on `net_investments`.

```cql
CREATE TABLE instance_status_by_strategy (strategy_id TEXT,
    instance_id TEXT,
    account_id TEXT,
    broker_id TEXT,
    last_update TIMESTAMP,
    net_investments DOUBLE,
    balance DOUBLE,
    equity DOUBLE,
    unrealized_profit DOUBLE,
    margin_used DOUBLE,
    margin_used_pct DOUBLE,
    margin_free DOUBLE,
    total_size DOUBLE,
    max_size DOUBLE,
    cnt_open_pos INT,
    cnt_open_trades INT,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    net_profit DOUBLE,
    net_open_positions_eur DOUBLE,
    gross_open_positions_eur DOUBLE,
    net_open_positions_usd DOUBLE,
    gross_open_positions_usd DOUBLE,
    exposures_map MAP<TEXT, DOUBLE>,
    PRIMARY KEY (strategy_id, instance_id))
WITH CLUSTERING ORDER BY (instance_id ASC);
```

### instance_status_by_account

- Find the last (most recent) `instance status` per `instance_id` for a given `account_id`.
- This is table is update by a Spark batch job based on `instance_status_by_strategy`.

```cql
CREATE TABLE instance_status_by_account (strategy_id TEXT,
    instance_id TEXT,
    account_id TEXT,
    broker_id TEXT static,
    last_update TIMESTAMP,
    net_investments DOUBLE,
    balance DOUBLE,
    equity DOUBLE,
    unrealized_profit DOUBLE,
    margin_used DOUBLE,
    margin_used_pct DOUBLE,
    margin_free DOUBLE,
    total_size DOUBLE,
    max_size DOUBLE,
    cnt_open_pos INT,
    cnt_open_trades INT,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    net_profit DOUBLE,
    net_open_positions_eur DOUBLE,
    gross_open_positions_eur DOUBLE,
    net_open_positions_usd DOUBLE,
    gross_open_positions_usd DOUBLE,
    exposures_map MAP<TEXT, DOUBLE>,
    PRIMARY KEY (account_id, instance_id))
WITH CLUSTERING ORDER BY (instance_id ASC);
```

Previous definition of the table as a *materialized view*:

```cql
CREATE MATERIALIZED VIEW instance_status_by_account as
SELECT *
FROM instance_status_by_strategy
PRIMARY KEY(account_id, instance_id);
```

### instance_returns_by_strategy

- Find the instance return results by `strategy_id` and order results by `account_id` ascending.
- Spark table created using tables `instance_status_by_strategy`, `closed_positions_by_strategy` and `transactions_by_account_type`

```cql
CREATE TABLE instance_returns_by_strategy (strategy_id TEXT,
    instance_id TEXT,
    account_id TEXT,
    inception_date TEXT,
    investments DOUBLE,
    withdrawals DOUBLE,
    net_investments DOUBLE,
    equity DOUBLE,
    ann_return DOUBLE,
    return_pct DOUBLE,
    PRIMARY KEY (strategy_id, instance_id))
WITH CLUSTERING ORDER BY (instance_id ASC);
```

### transactions_by_instance_type

- Find all the **transactions** by `instance_id`, transaction `type` and between an interval of **days**. Order by `tstamp` descending.
- The `date` has the format `YYYY-MM-DD`.

```cql
CREATE TABLE transactions_by_instance_type (type TEXT,
    instance_id TEXT,
    account_id TEXT,
    broker_id TEXT,
    strategy_id TEXT,
    date TEXT,
    tstamp TIMESTAMP,
    amount DOUBLE,
    transaction_id TEXT,
    comment TEXT,
    PRIMARY KEY ((instance_id, type), date, tstamp))
WITH CLUSTERING ORDER BY (date DESC, tstamp DESC);
```

### hourly_status_by_instance

- Find the hourly **instance status** by `instance_id` and ordered by `date_hour` (hour) ascending.
- The `date_hour` has the format `YYYY-MM-DD HH24`.
- This table is created by a spark job from table `instance_status_by_id_day`.

```cql
CREATE TABLE hourly_status_by_instance (instance_id TEXT,
    date_hour TEXT,
    account_id TEXT static,
    broker_id TEXT static,
    strategy_id TEXT static,
    last_update TIMESTAMP,
    balance DOUBLE,
    equity DOUBLE,
    unrealized_profit DOUBLE,
    margin_used DOUBLE,
    margin_used_pct DOUBLE,
    margin_free DOUBLE,
    total_size DOUBLE,
    max_size DOUBLE,
    cnt_open_pos INT,
    cnt_open_trades INT,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    net_profit DOUBLE,
    PRIMARY KEY (instance_id, date_hour))
WITH CLUSTERING ORDER BY (date_hour ASC);
```

### daily_status_by_instance

- Find the daily **instance status** by `instance_id` and ordered by `date` ascending.
- The `date` has the format `YYYY-MM-DD`.
- This table is created by a Spark job based on tables `instance_status_by_id_day` and `daily_transactions_by_instance_type` (required for `net_investments`).

```cql
CREATE TABLE daily_status_by_instance (instance_id TEXT,
    date TEXT,
    account_id TEXT static,
    broker_id TEXT static,
    strategy_id TEXT static,
    last_update TIMESTAMP,
    net_investments DOUBLE,
    balance DOUBLE,
    equity DOUBLE,
    unrealized_profit DOUBLE,
    margin_used DOUBLE,
    margin_used_pct DOUBLE,
    margin_free DOUBLE,
    total_size DOUBLE,
    max_size DOUBLE,
    cnt_open_pos INT,
    cnt_open_trades INT,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    net_profit DOUBLE,
    PRIMARY KEY (instance_id, date))
WITH CLUSTERING ORDER BY (date ASC);
```

### daily_transactions_by_instance_type

- Find the **transactions** grouped by `date` for a given `instance_id`, transaction `type` and between an interval of **days**. Order by `date` descending.
- The `date` has the format `YYYY-MM-DD`.
- This table is updated by a Spark batch job, that aggregates the transactions by day of the table `transactions_by_instance_type`.

```cql
CREATE TABLE daily_transactions_by_instance_type (instance_id TEXT,
    type TEXT,
    account_id TEXT static,
    broker_id TEXT static,
    strategy_id TEXT static,
    date TEXT,
    amount DOUBLE,
    PRIMARY KEY ((instance_id, type), date))
WITH CLUSTERING ORDER BY (date DESC);
```

### logs_by_instance

- Find the **logs** by `instance_id` and between and interval of **days**. Order by `date` and `tstamp` descending.
- The `date` has the format `YYYY-MM-DD`.

```cql
CREATE TABLE logs_by_instance (instance_id TEXT,
    date TEXT,
    tstamp TIMESTAMP,
    strategy_id TEXT static,
    broker_id TEXT static,
    account_id TEXT static,
    log_level TEXT,
    message TEXT,
    PRIMARY KEY (instance_id, date, tstamp))
WITH CLUSTERING ORDER BY (date DESC, tstamp DESC);
```

### daily_results_by_instance

- Find the daily results (profit, number of closed positions, ...) by `instance_id` and grouped by `date` (day). The results are ordered by `date` ascending.
- This table is updated by a Spark batch job that aggregates the information from the table `closed_positions_by_instance`.
- The `date` has the format `YYYY-MM-DD`.

```cql
CREATE TABLE daily_results_by_instance (instance_id TEXT,
    account_id TEXT static,
    broker_id TEXT static,
    strategy_id TEXT static,
    date TEXT,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    nbr_closed_positions INT,
    nbr_closed_trades INT,
    max_size INT,
    avg_profit DOUBLE,
    max_profit DOUBLE,
    PRIMARY KEY (instance_id, date))
WITH CLUSTERING ORDER BY (date ASC);
```

## 7. Risk Analysis

### margins_by_strategy

- Find the (calculated) margins information for all accounts, grouped by `broker_id` and `account_id`.
- This table is updated by a Spark batch job, that aggregates the data in the `open_positions_by_instance` and also obtains the `equity` and `margin_used_broker` information from the `account_status` table.

```cql
CREATE TABLE account_margins (account_id TEXT,
    broker_id TEXT,
    equity DOUBLE,
    margin_used_broker DOUBLE,
    margin_used_pct DOUBLE,
    margin_used DOUBLE,
    margin_next DOUBLE,
    margin_used_next DOUBLE,
    margin_next_pct DOUBLE,
    margin_used_next_pct DOUBLE,
    PRIMARY KEY (account_id));
```

## 8. Instruments

### instruments

- Find all instruments' symbols.
- This table will be migrated from the previous **MySQL** database.

```cql
CREATE TABLE instruments (symbol TEXT,
    point_scale INT,
    PRIMARY KEY (symbol));
```

### symbol_bias_by_day

- Find the total number of open positions, long and short positions, the `bias` and `div_index` for all **instruments** traded in all strategies for a specified `date`. Results ordered by `last_update` descending.
- The `date` has the format `YYYY-MM-DD`.

```cql
CREATE TABLE symbol_bias_by_day (date TEXT,
    last_update TIMESTAMP,
    symbol TEXT,
    nbr_open_trades INT,
    nbr_longs INT,
    nbr_shorts INT,
    long_units DOUBLE,
    short_units DOUBLE,
    bias DOUBLE,
    div_index DOUBLE,
    PRIMARY KEY ((date), last_update, symbol))
WITH CLUSTERING ORDER BY (last_update DESC, symbol ASC);
```

### symbol_bias_last_update

- Find the last updated *bias* for each symbol.

```cql
CREATE TABLE symbol_bias_last_update (symbol TEXT,
    last_update TIMESTAMP,
    nbr_open_trades INT,
    nbr_longs INT,
    nbr_shorts INT,
    long_units DOUBLE,
    short_units DOUBLE,
    bias DOUBLE,
    div_index DOUBLE,
    PRIMARY KEY (symbol));
```

### instruments_summary_by_strategy

- Find the historical total profit, average profit, number of trades and maximum size of all **instruments** traded in a given `strategy_id`.
- This table is updated by a Spark batch job from `closed_positions_by_strategy`.

```cql
CREATE TABLE instruments_summary_by_strategy (strategy_id TEXT,
    symbol TEXT,
    total_profit DOUBLE,
    avg_profit DOUBLE,
    nbr_closed_positions INT,
    max_size INT,
    PRIMARY KEY (strategy_id, symbol))
WITH CLUSTERING ORDER BY (symbol ASC);
```

### oanda_prices_by_symbol

- Find the **oanda prices** by `symbol` and between an interval between of dates, ordered by `date` ascending.
- The `date` has the format `YYYY-MM-DD`. 
- **TODO:** change the cronjob that currently uploads this data into the **MySQL** database.

```cql
CREATE TABLE oanda_prices_by_symbol (symbol TEXT,
    date TEXT,
    price DOUBLE,
    PRIMARY KEY (symbol, date))
WITH CLUSTERING ORDER BY (date ASC);
```

## 9. Closed Trades

### closed_trades_by_instance

- Find the **closed trades** by `instance_id` and order by `close_time` descending.

```cql
CREATE TABLE closed_trades_by_instance (instance_id TEXT,
    close_time TIMESTAMP,
    close_state TEXT,
    trade_id TEXT,
    position_id TEXT,
    broker_id TEXT,
    strategy_id TEXT,
    account_id TEXT,
    symbol TEXT,
    direction TEXT,
    size DOUBLE,
    open_time TIMESTAMP,
    duration DOUBLE,
    open_price DOUBLE,
    close_price DOUBLE,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    PRIMARY KEY (instance_id, close_time, trade_id))
WITH CLUSTERING ORDER BY (close_time DESC, trade_id ASC);
```

### closed_trades_by_account

- Find the **closed trades** by `account_id` and order by `close_time` descending.
- This table is updated by a Spark job based on `closed_trades_by_instance`.

```cql
CREATE TABLE closed_trades_by_account (instance_id TEXT,
    close_time TIMESTAMP,
    close_state TEXT,
    trade_id TEXT,
    position_id TEXT,
    broker_id TEXT,
    strategy_id TEXT,
    account_id TEXT,
    symbol TEXT,
    direction TEXT,
    size DOUBLE,
    open_time TIMESTAMP,
    duration DOUBLE,
    open_price DOUBLE,
    close_price DOUBLE,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    PRIMARY KEY (account_id, close_time, trade_id))
WITH CLUSTERING ORDER BY (close_time DESC, trade_id ASC);
```

Previous definition of the table as a *materialized view*:

```cql
CREATE MATERIALIZED VIEW closed_trades_by_account as
SELECT *
FROM closed_trades_by_instance
WHERE account_id IS NOT NULL
PRIMARY KEY(account_id, close_time, trade_id);
```

### closed_trades_by_strategy

- Find the **closed trades** by `strategy_id` and with a`closed_time` between an interval of dates. Order by `close_time` descending.
- This table is updated by a Spark job based on `closed_trades_by_instance`.

```cql
CREATE TABLE closed_trades_by_strategy (instance_id TEXT,
    close_time TIMESTAMP,
    close_state TEXT,
    trade_id TEXT,
    position_id TEXT,
    broker_id TEXT,
    strategy_id TEXT,
    account_id TEXT,
    symbol TEXT,
    direction TEXT,
    size DOUBLE,
    open_time TIMESTAMP,
    duration DOUBLE,
    open_price DOUBLE,
    close_price DOUBLE,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    PRIMARY KEY (strategy_id, close_time, trade_id))
WITH CLUSTERING ORDER BY (close_time DESC, trade_id ASC);
```

Previous definition of the table as a *materialized view*:

```cql
CREATE MATERIALIZED VIEW closed_trades_by_strategy as
SELECT *
FROM closed_trades_by_instance
WHERE strategy_id IS NOT NULL
PRIMARY KEY(strategy_id, trade_id);
```

### virtual_stops_by_strategy

- Find the **closed_trades** which have a `close_state = 'VIRTUAL_STOP'` for a  given `strategy_id`.
- This table is update by a Spark job based on  **closed_trades_by_strategy**.
- The column `close_state` is dropped because it's unnecessary.

```cql
CREATE TABLE virtual_stops_by_strategy (instance_id TEXT,
    close_time TIMESTAMP,
    trade_id TEXT,
    position_id TEXT,
    broker_id TEXT,
    strategy_id TEXT,
    account_id TEXT,
    symbol TEXT,
    direction TEXT,
    size DOUBLE,
    open_time TIMESTAMP,
    duration DOUBLE,
    open_price DOUBLE,
    close_price DOUBLE,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    PRIMARY KEY (strategy_id, close_time, trade_id))
WITH CLUSTERING ORDER BY (close_time DESC, trade_id ASC);
```

Previous definition of the table as a *materialized view*:

```cql
CREATE MATERIALIZED VIEW virtual_stops_by_strategy AS
SELECT *
FROM closed_trades_by_instance
WHERE close_state = 'VIRTUAL_STOP'
PRIMARY KEY(strategy_id, trade_id);
```

### last_week_closed_trades_by_instance

- Find the last week's **closed trades** by `instance_id` and order by `close_time` descending.
- This table is update by a Spark job.

```cql
CREATE TABLE last_week_closed_trades_by_instance (instance_id TEXT,
    close_time TIMESTAMP,
    close_state TEXT,
    trade_id TEXT,
    position_id TEXT,
    broker_id TEXT,
    strategy_id TEXT,
    account_id TEXT,
    symbol TEXT,
    direction TEXT,
    size DOUBLE,
    open_time TIMESTAMP,
    duration DOUBLE,
    open_price DOUBLE,
    close_price DOUBLE,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    PRIMARY KEY (instance_id, close_time, trade_id))
WITH CLUSTERING ORDER BY (close_time DESC, trade_id ASC);
```

## 10. Closed Positions

### closed_positions_by_instance

- Find the **closed positions** by `instance_id` and with a `closed_time` between an interval of dates. Order by `close_time` descending.
- This table is updated by a Spark batch job, that queries `closed_trades_by_instance` and groups the trades by `position_id`.

```cql
CREATE TABLE closed_positions_by_instance (instance_id TEXT,
    broker_id TEXT,
    strategy_id TEXT,
    account_id TEXT,
    close_time TIMESTAMP,
    position_id TEXT,
    symbol TEXT,
    direction TEXT,
    max_size DOUBLE,
    initial_size DOUBLE,
    nbr_trades INT,
    open_time TIMESTAMP,
    duration DOUBLE,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    PRIMARY KEY (instance_id, close_time, position_id))
WITH CLUSTERING ORDER BY (close_time DESC, position_id ASC);
```

### closed_positions_by_strategy

- Find the **closed positions** by `strategy_id` and with a `closed_time` between an interval of dates. Order by `close_time` descending.
- This table is update by a Spark job based on  `closed_positions_by_instance`.

```cql
CREATE TABLE closed_positions_by_strategy (instance_id TEXT,
    broker_id TEXT,
    strategy_id TEXT,
    account_id TEXT,
    close_time TIMESTAMP,
    position_id TEXT,
    symbol TEXT,
    direction TEXT,
    max_size DOUBLE,
    initial_size DOUBLE,
    nbr_trades INT,
    open_time TIMESTAMP,
    duration DOUBLE,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    PRIMARY KEY (strategy_id, close_time, instance_id, position_id))
WITH CLUSTERING ORDER BY (close_time DESC, instance_id ASC, position_id ASC);
```

Previous definition of the table as a *materialized view*:

```cql
CREATE MATERIALIZED VIEW closed_positions_by_strategy AS
SELECT *
FROM closed_positions_by_instance
PRIMARY KEY(strategy_id, instance_id, close_time, position_id);
```

### last_week_closed_positions_by_instance

- Find the **closed positions** by `instance_id` for the last 7 days (one rolling week). Order by `close_time` descending.
- This table is updated by a Spark batch job, that queries `closed_positions_by_instance` and extracts the positions in the specified period.

```cql
CREATE TABLE last_week_closed_positions_by_instance (instance_id TEXT,
    broker_id TEXT,
    strategy_id TEXT,
    account_id TEXT,
    close_time TIMESTAMP,
    position_id TEXT,
    symbol TEXT,
    direction TEXT,
    max_size DOUBLE,
    initial_size DOUBLE,
    nbr_trades INT,
    open_time TIMESTAMP,
    duration DOUBLE,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    PRIMARY KEY (instance_id, close_time, position_id))
WITH CLUSTERING ORDER BY (close_time DESC, position_id ASC);
```

## 11. Open Trades

### open_trades_by_instance

- Find all the **open trades** by `instance_id` and order by `trade_id` ascending.
- The columns `margin_required` and `margin_used` will be updated by a Spark job that queries the `ccy_required_margins_by_broker` and retrieves the `margin_required` for each `symbol` in a `broker_id`. The job will also obtain the currency symbol's last price/quote from the table **instruments**. Review how this table is updated.
- The columns `weight_pct`, `net_open_positons_eur/usd`, `gross_open_positions_eur/usd` will also  need to be updated by a Spark job.

```cql
CREATE TABLE open_trades_by_instance (strategy_id TEXT,
    trade_id TEXT,
    instance_id TEXT,
    account_id TEXT,
    broker_id TEXT,
    position_id TEXT,
    symbol TEXT,
    direction TEXT,
    size DOUBLE,
    open_time TIMESTAMP,
    duration DOUBLE,
    open_price DOUBLE,
    last_price DOUBLE,
    open_state TEXT,
    pips DOUBLE,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    units_base_ccy DOUBLE,
    units_quote_ccy DOUBLE,
    margin_required DOUBLE,
    margin_used DOUBLE,
    margin_next DOUBLE,
    margin_used_next DOUBLE,
    weight_pct DOUBLE,
    net_open_positions_eur DOUBLE,
    gross_open_positions_eur DOUBLE,
    net_open_positions_usd DOUBLE,
    gross_open_positions_usd DOUBLE,
    PRIMARY KEY (instance_id, trade_id))
WITH CLUSTERING ORDER BY (trade_id ASC);
```

### open_trades_by_strategy

- Find all the **open trades** by `strategy_id` and order by `trade_id` ascending.
- This table is updated by a Spark job based on `open_trades_by_instance`.

```cql
CREATE TABLE open_trades_by_strategy (strategy_id TEXT,
    trade_id TEXT,
    instance_id TEXT,
    account_id TEXT,
    broker_id TEXT,
    position_id TEXT,
    symbol TEXT,
    direction TEXT,
    size DOUBLE,
    open_time TIMESTAMP,
    duration DOUBLE,
    open_price DOUBLE,
    last_price DOUBLE,
    open_state TEXT,
    pips DOUBLE,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    units_base_ccy DOUBLE,
    units_quote_ccy DOUBLE,
    margin_required DOUBLE,
    margin_used DOUBLE,
    margin_next DOUBLE,
    margin_used_next DOUBLE,
    weight_pct DOUBLE,
    net_open_positions_eur DOUBLE,
    gross_open_positions_eur DOUBLE,
    net_open_positions_usd DOUBLE,
    gross_open_positions_usd DOUBLE,
    PRIMARY KEY (strategy_id, instance_id, trade_id))
WITH CLUSTERING ORDER BY (instance_id ASC, trade_id ASC);
```

Previous definition of the table as a *materialized view*:

```cql
CREATE MATERIALIZED VIEW open_trades_by_strategy AS
SELECT *
FROM open_trades_by_instance
WHERE strategy_id IS NOT NULL
PRIMARY KEY (strategy_id, instance_id, trade_id);
```

### open_trades_by_account

- Find all the **open trades** by `account_id` and order by `trade_id` ascending.
- This table is updated by a Spark job based on `open_trades_by_instance`.

```cql
CREATE TABLE open_trades_by_account (strategy_id TEXT,
    trade_id TEXT,
    instance_id TEXT,
    account_id TEXT,
    broker_id TEXT,
    position_id TEXT,
    symbol TEXT,
    direction TEXT,
    size DOUBLE,
    open_time TIMESTAMP,
    duration DOUBLE,
    open_price DOUBLE,
    last_price DOUBLE,
    open_state TEXT,
    pips DOUBLE,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    units_base_ccy DOUBLE,
    units_quote_ccy DOUBLE,
    margin_required DOUBLE,
    margin_used DOUBLE,
    margin_next DOUBLE,
    margin_used_next DOUBLE,
    weight_pct DOUBLE,
    net_open_positions_eur DOUBLE,
    gross_open_positions_eur DOUBLE,
    net_open_positions_usd DOUBLE,
    gross_open_positions_usd DOUBLE,
    PRIMARY KEY (account_id, instance_id, trade_id))
WITH CLUSTERING ORDER BY (instance_id ASC, trade_id ASC);
```

Previous definition of the table as a *materialized view*:

```cql
CREATE MATERIALIZED VIEW open_trades_by_account AS
SELECT *
FROM open_trades_by_instance
WHERE instance_id IS NOT NULL
PRIMARY KEY (account_id, instance_id, trade_id);
```

## 12. Open Positions

### open_positions_by_instance

- Find all the **open positions** by `instance_id` and orders them by `position_id` ascending.
- This table is updated by a Spark batch job, that queries `open_trades_by_instance` and groups the trades by `instance_id`.

```cql
CREATE TABLE open_positions_by_instance (strategy_id TEXT,
    instance_id TEXT,
    position_id TEXT,
    account_id TEXT,
    broker_id TEXT,
    symbol TEXT,
    direction TEXT,
    max_size DOUBLE,
    size DOUBLE,
    nbr_trades DOUBLE,
    open_time TIMESTAMP,
    duration DOUBLE,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    initial_size DOUBLE,
    expected_profit DOUBLE,
    units_base_ccy DOUBLE,
    units_quote_ccy DOUBLE,
    margin_used DOUBLE,
    margin_next DOUBLE,
    margin_used_next DOUBLE,
    weight_pct DOUBLE,
    net_open_positions_eur DOUBLE,
    gross_open_positions_eur DOUBLE,
    net_open_positions_usd DOUBLE,
    gross_open_positions_usd DOUBLE,
    PRIMARY KEY (instance_id, position_id))
WITH CLUSTERING ORDER BY (position_id ASC);
```

### open_positions_by_strategy

- Find all the **open positions** by `strategy_id`.
- This table is updated by a Spark job based on `open_positions_by_instance`.

```cql
CREATE TABLE open_positions_by_strategy (strategy_id TEXT,
    instance_id TEXT,
    position_id TEXT,
    account_id TEXT,
    broker_id TEXT,
    symbol TEXT,
    direction TEXT,
    max_size DOUBLE,
    size DOUBLE,
    nbr_trades DOUBLE,
    open_time TIMESTAMP,
    duration DOUBLE,
    commission DOUBLE,
    swap DOUBLE,
    gross_profit DOUBLE,
    profit DOUBLE,
    initial_size DOUBLE,
    expected_profit DOUBLE,
    units_base_ccy DOUBLE,
    units_quote_ccy DOUBLE,
    margin_used DOUBLE,
    margin_next DOUBLE,
    margin_used_next DOUBLE,
    weight_pct DOUBLE,
    net_open_positions_eur DOUBLE,
    gross_open_positions_eur DOUBLE,
    net_open_positions_usd DOUBLE,
    gross_open_positions_usd DOUBLE,
    PRIMARY KEY (strategy_id, instance_id, position_id))
WITH CLUSTERING ORDER BY (instance_id ASC, position_id ASC);
```

Previous definition of the table as a *materialized view*:

```cql
CREATE MATERIALIZED VIEW open_positions_by_strategy AS
SELECT *
FROM open_positions_by_instance
WHERE strategy_id IS NOT NULL
PRIMARY KEY (strategy_id, instance_id, position_id);
```

## 13. Live environment

### critical_variables_by_instance_id_instrument

- Obtain all critical varible values from a running instance.

```cql
CREATE TABLE critical_variables_by_instance_id_instrument (instance_id TEXT,
    instrument TEXT,
    strategy_id TEXT,
    critical_variables MAP<TEXT, TEXT>,
    last_update TIMESTAMP,
    PRIMARY KEY (instance_id, instrument));
```

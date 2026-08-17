# PostgreSQL Decision Logger

### Persistent Storage for LLM Agent Simulation Runs

A database-backed logging layer for a **Supply Chain Digital Twin** simulation. The project persists agent decisions, daily simulation states, and key performance indicators (KPIs) so that supply chain experiments can be reproduced, compared, and analysed across multiple simulation runs.

The implementation supports both **SQLite** for local/Colab experimentation and **PostgreSQL** for deployment on a real database server such as a university database or Supabase.

---

## Overview

The simulation models a supply chain consisting of:

* Customer agents generating daily demand
* A supplier agent managing capacity and delayed shipments
* An LLM-inspired heuristic retailer making inventory and ordering decisions
* Different operational objectives such as **cost**, **availability**, and **COVID/disruption resilience**

Each simulation run is persisted in a relational database.

The system stores:

1. Simulation-level configuration
2. Daily inventory and demand states
3. Agent decision reasoning
4. Run-level KPIs

This makes it possible to analyse not only **what happened**, but also **why the agent made each decision**.

---

## Key Features

* Persistent simulation logging
* SQLite and PostgreSQL support
* SQLAlchemy-based database abstraction
* Automatic database schema creation
* Daily state tracking
* Agent decision/reasoning logging
* KPI calculation and storage
* Scenario comparison
* Stockout analysis
* Ordering behaviour analysis
* Bullwhip ratio analysis
* Database-driven visualisation
* CSV export for further analysis
* Reproducible simulations using random seeds
* PostgreSQL deployment support

---

## System Architecture

```text
Supply Chain Simulation
        │
        ├── Customer Agents
        │       └── Generate daily demand
        │
        ├── Supplier Agent
        │       └── Handles capacity and lead time
        │
        └── Retailer Agent
                └── Makes inventory/order decisions
                         │
                         ▼
                 SQLAlchemy Engine
                         │
              ┌──────────┴──────────┐
              │                     │
           SQLite              PostgreSQL
              │                     │
              └──────────┬──────────┘
                         ▼
                  Persistent Data
                         │
             ┌───────────┼───────────┐
             ▼           ▼           ▼
        Daily States    KPIs     Decision Logs
             │           │           │
             └───────────┼───────────┘
                         ▼
                  Analysis & Export
```

---

## Database Schema

The database consists of three main tables.

### 1. `simulation_runs`

Stores metadata for each simulation run.

| Column       | Description                      |
| ------------ | -------------------------------- |
| `run_id`     | Unique simulation run identifier |
| `scenario`   | Scenario name                    |
| `agent_type` | Agent implementation used        |
| `objective`  | Decision objective               |
| `lead_time`  | Supplier lead time               |
| `seed`       | Random seed                      |
| `steps`      | Number of simulation steps       |
| `created_at` | Run creation timestamp           |

---

### 2. `daily_states`

Stores the state of the simulation for every day.

| Column             | Description                |
| ------------------ | -------------------------- |
| `run_id`           | Associated simulation run  |
| `day`              | Simulation day             |
| `inventory`        | Current inventory          |
| `backlog`          | Unfulfilled demand         |
| `demand`           | Daily demand               |
| `order_qty`        | Quantity ordered           |
| `holding_cost_cum` | Cumulative holding cost    |
| `backlog_cost_cum` | Cumulative backlog cost    |
| `reasoning`        | Agent's decision reasoning |

The `reasoning` field allows the simulation to preserve the decision context alongside the numerical state.

---

### 3. `run_kpis`

Stores aggregated performance metrics for each run.

| KPI               | Description                                 |
| ----------------- | ------------------------------------------- |
| Service Level     | Percentage of days without backlog          |
| Fill Rate         | Proportion of demand successfully fulfilled |
| Average Inventory | Mean inventory level                        |
| Average Backlog   | Mean backlog level                          |
| Total Cost        | Holding cost + backlog cost                 |
| Order Frequency   | Percentage of days with an order            |
| Bullwhip Ratio    | Order-demand variability ratio              |

---

## Simulation Scenarios

Four scenarios are evaluated for 90 simulation steps.

### 1. Normal — Cost Focus

```text
Objective: cost
Lead time: 3 days
Supplier capacity: 600
Seed: 42
```

The retailer prioritises cost efficiency under normal operating conditions.

### 2. Normal — Availability

```text
Objective: availability
Lead time: 3 days
Supplier capacity: 600
Seed: 42
```

The retailer prioritises product availability and service level.

### 3. Disruption — Long Lead Time

```text
Objective: availability
Lead time: 7 days
Supplier capacity: 600
Seed: 42
```

This scenario evaluates the effect of a longer supply lead time.

### 4. COVID — Shock + Disruption

```text
Objective: covid
Lead time: 7 days
Supplier capacity: 600

Demand shock:
Day 30
Shock multiplier: 2.5

Supplier disruption:
Day 35
Disrupted capacity: 150
```

This scenario combines a sudden demand increase with a supplier capacity disruption.

---

## Example Results

The four simulation runs produced the following KPI results:

| Scenario                 | Service Level | Fill Rate | Avg. Inventory | Total Cost | Bullwhip Ratio |
| ------------------------ | ------------: | --------: | -------------: | ---------: | -------------: |
| Normal - Cost Focus      |         98.9% |    100.0% |          278.6 |    $25,089 |         10.189 |
| Normal - Availability    |        100.0% |    100.0% |          290.7 |    $26,166 |         10.098 |
| COVID - Shock+Disruption |         44.4% |     90.0% |          236.0 |    $42,081 |          1.872 |
| Disruption - Long LT     |         96.7% |     97.1% |        2,167.1 |   $197,727 |         23.148 |

The results show a substantial difference between normal and disrupted operating conditions.

The **Normal - Cost Focus** scenario achieved the lowest total cost, while the **Normal - Availability** scenario achieved the highest service level.

The **Disruption - Long LT** scenario produced a very high average inventory and total cost, together with the highest bullwhip ratio.

The **COVID - Shock+Disruption** scenario experienced the lowest service level due to the combined demand shock and supplier disruption.

---

## Stockout Analysis

The database can be queried to identify the most severe stockout days across all simulation runs.

Example query:

```sql
SELECT
    r.scenario,
    d.day,
    d.backlog,
    d.demand,
    d.inventory,
    d.reasoning
FROM daily_states d
JOIN simulation_runs r
    ON d.run_id = r.run_id
WHERE d.backlog > 0
ORDER BY d.backlog DESC
LIMIT 10;
```

The most severe stockout occurred in the **COVID - Shock+Disruption** scenario:

```text
Day: 32
Backlog: 521
Demand: 271
Inventory: 0
Order: 240
```

The stored reasoning was:

```text
[covid] inv=0 target=1654 order=240
```

This demonstrates how the database connects the observed operational outcome with the agent's decision context.

---

## Ordering Behaviour Analysis

The system also analyses ordering behaviour by scenario.

Example metrics include:

* Number of ordering days
* Average order size
* Maximum order size
* Average inventory

The **Disruption - Long LT** scenario showed the strongest ordering response:

```text
Days ordered: 20
Average order: 518.2
Maximum order: 1073
Average inventory: 2167.1
```

This behaviour can be used to investigate potential **panic-buying and bullwhip effects** caused by longer lead times.

---

## Outputs

The notebook generates persistent and analytical outputs.

### SQLite Database

```text
outputs/supply_chain_sim.db
```

Contains:

* Simulation runs
* Daily states
* Agent reasoning
* KPIs

### Trajectory Plot

```text
outputs/db_trajectories.png
```

Visualises inventory and backlog trajectories retrieved directly from the database.

### Full Simulation Dataset

```text
outputs/full_simulation_data.csv
```

The current experiment exports:

```text
360 rows
4 simulation runs
90 days per run
```

The CSV includes simulation configuration, daily state variables, cumulative costs, and agent reasoning.

---

## Installation

Install the required Python packages:

```bash
pip install psycopg2-binary sqlalchemy mesa pandas matplotlib
```

For Google Colab, the notebook uses:

```bash
pip install psycopg2-binary sqlalchemy mesa pandas matplotlib --break-system-packages -q
```

---

## Database Configuration

By default, the project uses SQLite:

```python
USE_SQLITE = True
```

The SQLite database is created automatically:

```text
outputs/supply_chain_sim.db
```

No external database server is required.

---

## Using PostgreSQL

To connect to a real PostgreSQL database, change:

```python
USE_SQLITE = False
```

Then configure:

```python
PG_HOST = "localhost"
PG_PORT = 5432
PG_DATABASE = "supply_chain"
PG_USER = "postgres"
```

The password should be provided through an environment variable rather than hardcoded:

```python
import os
os.environ["PG_PASSWORD"] = "your_password"
```

The connection URL is then generated automatically using SQLAlchemy.

The same simulation and logging code can be used for both SQLite and PostgreSQL. Only the database connection configuration changes.

---


## Reproducibility

Simulation runs use explicit random seeds.

Example:

```python
seed = 42
```

This allows experiments to be repeated under the same stochastic conditions and makes scenario comparisons more consistent.

Each run also stores its:

* Scenario
* Agent type
* Objective
* Lead time
* Random seed
* Number of simulation steps

in the `simulation_runs` table.

## Technologies

* **Python**
* **Mesa** — agent-based simulation
* **SQLAlchemy** — database abstraction
* **SQLite** — local persistence
* **PostgreSQL** — production/remote persistence
* **Pandas** — data analysis
* **Matplotlib** — visualisation
* **psycopg2** — PostgreSQL connectivity
* **Supabase** — optional cloud PostgreSQL backend

---

## Future Extensions

Potential extensions include:

* Additional LLM-based decision agents
* More realistic supplier networks
* Multiple retailers and warehouses
* Dynamic pricing
* Multi-echelon supply chains
* More disruption scenarios
* Agent-to-agent communication
* LLM-generated decision explanations
* Experiment dashboards
* Automated scenario benchmarking
* Cloud-hosted experiment tracking


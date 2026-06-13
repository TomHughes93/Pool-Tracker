# Cloud-Connected Sports Analytics Engine: Mobile Field Tracker to Power BI

## Executive Summary

In amateur sports tracking, field data collection is traditionally plagued by friction. Manual paper tallies, clunky spreadsheets, and generic scoring applications fail in high-frequency, real-world environments like a busy pub. This project resolves that operational friction by deploying a custom, mobile-first web application engineered specifically for quick, high-impact touch logging.

The edge application functions as a live data collection tool, transmitting transactional game metrics directly to a hosted PostgreSQL cloud data warehouse upon frame completion. This data pipeline feeds an analytical Power BI dashboard, converting raw table actions into structured performance diagnostics, player form trends, and tactical error analysis.

---

## System Architecture & Data Pipeline

The end-to-end data pipeline is structured into three decoupled layers to ensure scalability and data integrity:

```text
[ Edge App: HTML5/JS ] ---> ( REST API Gateway ) ---> [ Cloud DB: Supabase / Postgres ] ---> [ BI Layer: Power BI Desktop ]

```

1. **Data Collection Layer (Edge):** A lightweight HTML5 web application optimized for mobile viewports. It handles active frame state machine logic locally and utilizes browser `localStorage` cache fallbacks to ensure zero data loss during temporary network drops.
2. **Cloud Storage Layer (Warehouse):** An authenticated REST API gateway transfers the data payloads directly into a hosted Supabase PostgreSQL instance. Database integrity is protected by structural constraints, index configurations, and strict token validation.
3. **Analytics & Visualization Layer (BI):** Power BI Desktop establishes a secure connection to the cloud database, executing automated transformation scripts via Power Query to model the metrics into user-facing executive dashboards.

---

## Edge Application Logic & Embedded State Machine

The core challenge of tracking a live match is managing the ruleset seamlessly in the background so the user only has to tap plain results. The application handles this by running an embedded JavaScript state machine that actively computes player states before packaging the database payload.

### 1. Dynamic Match Open State

When a match initiates, the table is flags as `tableColorsAssigned = false`. The application monitors the break actions via a specialized tracking buffer:

* Pots executed using the break adjustment tools (`breakPotAdjust()`) increment the `break_pots` count for the active player but maintain the open table state.
* The moment a standard shot is logged, the system executes an automated check. If a color is claimed, it calls `assignTableColors()`, updating the UI indicators and mapping the correct colors to each player object to lock down subsequent ball deductions.

### 2. The Multi-Visit Foul Logic

Handling fouls in amateur pool requires tracking state across multiple turns. The application implements this using a strict `visitsRemaining` state tracker inside `logShot()`:

* **Standard Turn:** A player begins their turn with `visitsRemaining = 1`. A miss toggles the active player and resets visits to 1.
* **Foul Incurred:** When any of the three foul triggers are tapped (White scratch, No Hit, or Opponent Pot), the system increments the active player's `total_fouls` metric, switches control to the opponent, and aggressively sets `visitsRemaining = 2`.
* **Foul Turn Management:** On a foul visit, if the incoming player misses on their first attempt, the logic intercepts the standard player switch. It decrements `visitsRemaining` down to 1 but keeps that same player active at the table. Control only switches over if they miss a second consecutive time.

### 3. Transactional History Stack (The Undo Engine)

To prevent pub distractions from corrupting data integrity, the app utilizes a strict LIFO (Last-In, First-Out) stack array named `actionHistoryStack`.

* Before any state variable is updated by a button tap, `captureHistorySnapshot()` deep-serializes the entire state of the game (current remaining balls, scores, active player flags, and deep-copied statistics objects) into a JSON string and pushes it to the stack.
* Tapping the square `<= Undo` button pops the top snapshot off the stack, parses the variables back into live memory, and re-renders the UI elements, maintaining perfect data synchronization.

---

## Relational Database Configuration (PostgreSQL / SQL)

The cloud storage layer is powered by a PostgreSQL database hosted on Supabase. Below are the production-grade SQL scripts used to establish the data warehouse architecture, enforce relational constraints, and build analytical data layers.

### 1. Base Table Creation: `match_summaries`

This script defines the physical storage schema, enforcing strict non-nullable fields on key metrics to prevent null-pointer calculations during BI ingestion.

```sql
CREATE TABLE public.match_summaries (
    id BIGINT GENERATED BY DEFAULT AS IDENTITY,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    match_id TEXT NOT NULL,
    player_name TEXT NOT NULL,
    is_winner BOOLEAN NOT NULL,
    total_shots INTEGER DEFAULT 0 NOT NULL,
    total_misses INTEGER DEFAULT 0 NOT NULL,
    total_fouls INTEGER DEFAULT 0 NOT NULL,
    break_pots INTEGER DEFAULT 0 NOT NULL,
    break_misses INTEGER DEFAULT 0 NOT NULL,
    
    CONSTRAINT match_summaries_pkey PRIMARY KEY (id)
);

-- Index optimization for rapid analytical filtering across player dimensions
CREATE INDEX idx_match_summaries_player ON public.match_summaries (player_name);
CREATE INDEX idx_match_summaries_match ON public.match_summaries (match_id);

```

### 2. Embedded Server-Side Views (Aggregated Metrics)

To showcase true database layer engineering, we configured an optimized server-side view. This allows thin reporting clients or analytical queries to pull pre-aggregated performance data directly from the server without wasting processing power recalculating rows at the application level.

```sql
CREATE OR REPLACE VIEW public.vw_player_career_metrics AS
SELECT 
    player_name,
    COUNT(id) AS total_frames_played,
    COUNT(id) FILTER (WHERE is_winner = TRUE) AS total_wins,
    COUNT(id) FILTER (WHERE is_winner = FALSE) AS total_losses,
    
    -- Win Percentage Calculation
    ROUND(
        (COUNT(id) FILTER (WHERE is_winner = TRUE))::NUMERIC / COUNT(id)::NUMERIC * 100, 
        1
    ) AS win_percentage,
    
    SUM(total_shots) AS career_shots_taken,
    SUM(total_misses) AS career_misses,
    
    -- Potting Accuracy Calculation
    ROUND(
        (SUM(total_shots) - SUM(total_misses))::NUMERIC / SUM(total_shots)::NUMERIC * 100, 
        1
    ) AS career_potting_accuracy,
    
    SUM(total_fouls) AS career_fouls_committed,
    
    -- Foul Frequency per Shot Taken
    ROUND(
        SUM(total_fouls)::NUMERIC / SUM(total_shots)::NUMERIC, 
        3
    ) AS fouls_per_shot,
    
    SUM(break_pots) AS career_break_pots,
    SUM(break_misses) AS career_break_misses,
    
    -- Break Success Rate
    ROUND(
        SUM(break_pots)::NUMERIC / (SUM(break_pots) + SUM(break_misses))::NUMERIC * 100, 
        1
    ) AS break_efficiency
FROM 
    public.match_summaries
GROUP BY 
    player_name;

```

---

## Power BI Data Model & DAX Design

When pulling data straight from the PostgreSQL cloud instance into Power BI, the application establishes a formal star schema model. A dedicated time dimension table (`Dim_Calendar`) is generated via DAX to isolate temporal tracking from raw logs.

### Dim_Calendar Generation

```dax
Dim_Calendar = 
VAR BaseCalendar = CALENDAR(MIN(match_summaries[created_at]), MAX(match_summaries[created_at]))
RETURN
ADDCOLUMNS(
    BaseCalendar,
    "Year", YEAR([Date]),
    "Month", FORMAT([Date], "MMMM"),
    "MonthSort", MONTH([Date]),
    "WeekNumber", WEEKNUM([Date]),
    "DayOfWeek", FORMAT([Date], "DDDD")
)

```

### Core Business Intelligence Measures

To evaluate actual player quality during performance analysis, the data model converts the raw fact columns into precise analytical rates using Data Analysis Expressions (DAX):

#### Potting Accuracy Percentage

Determines overall shot precision by calculating successful pots against total shot volume.

```dax
Potting Accuracy = 
DIVIDE(
    SUM(match_summaries[total_shots]) - SUM(match_summaries[total_misses]), 
    SUM(match_summaries[total_shots]), 
    0
)

```

#### Pots Per Visit Average

Measures offensive run efficiency by dividing successful clearances by total defensive turnovers (misses) and frame completions (wins).

```dax
Pots Per Visit = 
DIVIDE(
    SUM(match_summaries[total_shots]) - SUM(match_summaries[total_misses]), 
    SUM(match_summaries[total_misses]) + CALCULATE(COUNTROWS(match_summaries), match_summaries[is_winner] = TRUE), 
    0
)

```

#### Average Visits Per Frame

Establishes match pace and defensive tightness by measuring the average number of table visits required to conclude a frame.

```dax
Avg Visits Per Frame = 
DIVIDE(
    SUM(match_summaries[total_misses]) + CALCULATE(COUNTROWS(match_summaries), match_summaries[is_winner] = TRUE),
    CALCULATE(COUNTROWS(match_summaries), match_summaries[is_winner] = TRUE),
    0
)

```

---

## Portfolio Presentation Blueprint

When presenting this complete architecture to hiring managers or technical interviewers, structure your project walkthrough around these three software engineering pillars:

* **Edge State Reliability:** Explain how you built a robust, real-world state machine in vanilla JavaScript. Walk through the logic that accurately manages multi-visit foul structures and a LIFO memory stack to handle real-time user errors or connectivity drops seamlessly.
* **Full-Stack Pipeline Engineering:** Highlight that you avoided the amateur approach of using intermediate csv files. Demonstrate how you configured a production-ready cloud pipeline that uses web interfaces to write clean JSON data directly to a hosted SQL engine via an API gateway.
* **Database Optimization vs Model Translation:** Prove your understanding of efficient query architecture. Explain why you implemented server-side SQL views for heavy, high-level group aggregations, while utilizing client-side Power BI DAX modeling for complex filter-context variables like dynamic head-to-head ratios.

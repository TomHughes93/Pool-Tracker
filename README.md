# Project Documentation: Pool Tracker Application

A lightweight, mobile-first web application designed to track real-time pool match statistics and stream granular shot-by-shot analytics directly to a Supabase PostgreSQL database for Business Intelligence (BI) reporting.

---

## Technical Architecture Overview

The system is engineered as a decoupled, serverless application. It prioritizes instant UI responsiveness on mobile browsers while handling data operations asynchronously to ensure performance stability even under fluctuating network conditions.

### 1. Unified Relational Database Schema
All gameplay events are streamed into a single unified table layout (`public.pool_shots`). This architecture captures granular match progressions and final states within a single flat dataset, optimizing query speeds for analytical aggregation.

* **`shot_number` (int4):** Incremental sequence tracker for actions within an isolated frame.
* **`match_id` (text):** Dynamically generated unique string (`match_EpochTime`) grouping all rows belonging to a single frame.
* **`player_1_home` / `player_2_away` (text):** Selected names of the competing players.
* **`active_player` (text):** The player executing the current shot layout.
* **`shot_result` (text):** Categorized outcome labels capturing precise actions:
  * Gameplay: `Pot`, `Miss`
  * Distinguishable Foul Metrics: `Foul - Potted White`, `Foul - No Hit`, `Foul - Opponent Potted`
  * Definitive Frame States: `Pot Black (Win Game)`, `Loss of Frame - Illegal Early Black Pot`
* **`visits_at_shot` (int4):** Remaining table visits available to the active player (1 or 2).
* **`dataset_type` (text):** Descriptive metadata logging the initial frame context (`Breaker: PlayerName`).
* **`match_result` (text):** State flag fixed to `In Progress` during active frames, updating to a definitive summary string (`Winner: Name | Loser: Name`) only on the final shot row.
* **`timestamp` (timestamptz):** ISO-8601 server-stamped execution time.

---

## Analytical Capabilities & BI View Generation

The flat-table layout allows you to generate distinct, high-level business intelligence layers using SQL views without duplicating raw files or altering your base table storage.

### Relational Match Summary View
Run this script within the Supabase SQL Editor to generate a permanent relational view (`view_match_summaries`) that isolates overall match history, breaker configurations, and definitive wins or losses:

```sql
CREATE OR REPLACE VIEW public.view_match_summaries AS
SELECT 
    match_id,
    timestamp::date AS match_date,
    player_1_home,
    player_2_away,
    REPLACE(dataset_type, 'Breaker: ', '') AS breaker,
    CASE 
        WHEN match_result LIKE '%Winner: %' THEN 
            SPLIT_PART(SPLIT_PART(match_result, 'Winner: ', 2), ' | ', 1)
        ELSE NULL 
    END AS winner,
    CASE 
        WHEN match_result LIKE '%Loser: %' THEN 
            SPLIT_PART(match_result, 'Loser: ', 2)
        ELSE NULL 
    END AS loser,
    CASE 
        WHEN match_result LIKE '%Loss of Frame%' THEN 'Foul Violation'
        ELSE 'Clearance'
    END AS win_reason
FROM public.pool_shots
WHERE match_result != 'In Progress';

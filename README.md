# Pool-Tracker
Project Portfolio: Mobile Pool Tracker Infrastructure
Live Artifacts & Deployment Links

    Production Application URL: https://tomhughes93.github.io/pool-tracker/

    Cloud Data Warehouse Endpoint: https://dwpwxywkexjxedkgvgyi.supabase.co

    Source Code Repository: Hosted publicly via GitHub Pages (integrated version control management).
Project Overview

The objective was to design, develop, and deploy a mobile-optimized sports analytics application capable of tracking real-time match events during Old EPA rule pool frames. The system eliminates manual logging by converting real-world athletic interactions into production-grade JSON strings, streaming them over a cloud API pipeline directly into a live relational database for performance analysis.
Technical Milestones & Architecture
1. Rules Engine & State Management (JavaScript)

    Designed a deterministic state machine using vanilla JavaScript to handle the complex penalty structures of Old EPA pool rules.

    Engineered a conditional logic loop that monitors a user's remaining visits. The engine automatically handles complex state transitions, such as downgrading a two-visit foul penalty to a single visit if a player misses their target ball on the opening shot of a turn, or passing control to the opposing player once all visits are exhausted.

2. Mobile User Interface & View Optimization

    Built a minimal, thumb-target interface with a responsive layout designed for one-handed mobile use in real-world environments.

    Implemented hidden toggle structures that allow the user to minimize setup and live data blocks during a frame, focusing screen real estate entirely on high-frequency interaction buttons.

3. Data Schema Design & Cloud API Integration

    Modelled an event-driven relational database schema inside an enterprise PostgreSQL instance hosted on Supabase.

    Created a table structure consisting of numeric identifiers, text properties for dynamic player profiling, and high-precision timestamptz properties to capture historical event logs accurately.

    Integrated an asynchronous network pipeline utilizing the modern JavaScript fetch() API. The engine converts the short-term memory array into an optimized JSON payload and posts it wirelessly via secure headers directly to the database endpoint upon match completion.

4. Resiliency & Local Cache Fallbacks

    Solved critical real-world mobile problems (pub signal dropouts, screen timeout refreshes) by integrating the browser's permanent memory cache (localStorage).

    Structured a recovery engine that backs up the live match state on every button click. If a browser tab reloads mid-game, the app parses the local cache on page load and recovers the frame seamlessly.

    Introduced a unique match_id variable derived from the ISO timestamp string to prevent concurrent data streams from overlapping within the central SQL warehouse.

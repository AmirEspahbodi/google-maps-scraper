# Google Maps Scraper

## Overview
This repository contains a Google Maps scraping system composed of two cooperating services:

- **Scraper worker**: Uses Playwright to open multiple browser contexts/tabs, search Google Maps, scroll listings, extract details, and persist results to Excel (and optionally download listing images). The worker pulls work items from a Redis queue. 
- **API server**: A FastAPI service that accepts scrape requests, validates duplicates against Redis queues and stored output files, and exposes a status endpoint to inspect current queue and processing state.

A helper script (`run_app.py`) runs both services concurrently using a thread pool.

## Key Features
- Asynchronous Playwright-driven scraping with multiple browser tabs/contexts.
- Redis-backed queue for search requests and in-process tracking.
- Excel output via Pandas + OpenPyXL.
- Optional image download and storage.
- FastAPI endpoints for requesting scrapes and observing system status.

## Tech Stack
- **Python 3.12** (Poetry-managed dependencies). 
- **Playwright** for browser automation. 
- **FastAPI** + **Uvicorn** for the API server. 
- **Redis** for queueing and coordination. 
- **Pydantic Settings** for environment configuration. 
- **lxml** for HTML parsing. 
- **Pandas / OpenPyXL** for Excel output. 
- **aiofiles / httpx** for async file and HTTP operations.

## Architecture & Data Flow
1. The API server receives a scrape request (listing category + query details) and enqueues it in Redis. 
2. The scraper worker reads queued requests, launches Playwright browsers/tabs, navigates to Google Maps, and performs search + scrolling. 
3. Listings are extracted, deduplicated across tabs, and enriched with details (title, category, address, phone, website, hours, coordinates, images). 
4. Results are written to Excel files in the configured storage directory.

## Design Patterns & Structure
- **BO/DAO/DTO layering**:
  - `data/bo`: Business logic orchestration for search, scrolling, extraction, and multi-tab coordination.
  - `data/dao`: Infrastructure wrappers (Redis access, Playwright page wrapper).
  - `data/dto`: Request/response schemas for the API.
- **Singleton RuntimeResource**: Centralized Playwright lifecycle management for browsers and tabs.
- **Async concurrency**: Heavy use of `asyncio.gather` for multi-tab scraping and page interactions.

## Project Structure
```
.
├── run_app.py                # Runs scraper + API server together
├── scraper/
│   ├── run_scraper.py         # Entry point for the scraper worker
│   ├── processes/             # Scraper processes
│   ├── data/                  # bo/dao/dto + static data (user agents, referers)
│   ├── config/                # Pydantic settings for scraper + Redis
│   └── utils/                 # Helper utilities (Excel, image storage, singleton)
└── server/
    ├── main.py                # API server entry point
    ├── apis/                  # FastAPI routes + schemas
    ├── core/                  # Server configuration + Redis connection
    └── utils/                 # File-based request status helpers
```

## Requirements
- Python 3.12
- Redis (local or remote)
- Playwright browsers installed (Chromium/WebKit)

## Setup
1. **Install dependencies**
   ```bash
   poetry install
   ```
2. **Install Playwright browsers**
   ```bash
   poetry run playwright install
   ```
3. **Configure environment**
   ```bash
   cp .env.sample .env
   ```
   Update values as needed (Redis host/port, storage directories, separators, etc.).

## Running the System
### Run both services together
```bash
poetry run python run_app.py
```

### Run scraper only
```bash
cd scraper
poetry run python run_scraper.py
```

### Run API server only
```bash
cd server
poetry run python main.py
```

## API Endpoints
### `POST /request`
Queues a new scrape request.

**Request body**
```json
{
  "listing_category": "restaurant",
  "listing_type": "restaurant",
  "verb": "near",
  "city": "Berlin",
  "province": "Berlin"
}
```

**Response**
Returns lists of imported, not-imported, in-process, and waiting requests.

### `GET /status`
Returns current status:
- `imported_requests` and `not_imported_requests` (based on storage files)
- `in_process` (current Redis processing item)
- `waiting_to_scrape` (queue contents)

## Output & Storage
- **Excel files**: Written to `NOT_IMPORTED_SHEETS_DIRECTORY` with a filename derived from the query fields and separators.
- **Images**: Saved under `PICTURES_DIRECTORY` (if image URLs are found).

## Configuration Reference
Environment variables are defined in `.env.sample` and loaded via Pydantic Settings. Key variables include:
- `REDIS_HOST`, `REDIS_PORT`
- `REDIS_REQUESTED_SEARCH_QUERY_QUEUE_NAME`, `REDIS_IN_PROCESSING_SEARCH_QUERY`
- `SEARCH_QUERY_ITEMS_SEPARATOR`, `LISTING_TYPE_ITEMS_SEPARATOR`
- `PICTURES_DIRECTORY`, `NOT_IMPORTED_SHEETS_DIRECTORY`, `IMPORTED_SHEETS_DIRECTORY`
- API server settings: `APP_NAME`, `HOST`, `PORT`, `ENVIRONMENT`, `WORKER`

## Notes & Considerations
- Scraping Google Maps may violate Google’s terms of service. Use responsibly and ensure compliance with local laws and platform policies.
- Playwright is configured to run in non-headless mode by default in `RuntimeResource`.
- Redis queues use simple list operations; scaling or fault tolerance might require additional coordination logic.

## License
This project is licensed under the terms of the MIT license. See [LICENSE](LICENSE).

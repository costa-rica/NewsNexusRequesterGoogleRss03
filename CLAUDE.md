# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

NewsNexusRequesterGoogleRss03 is an automated news aggregation system that fetches articles from Google News RSS feeds based on search queries defined in an Excel spreadsheet. The application intelligently prioritizes requests, tracks request history in a database, and processes results through a semantic scoring pipeline.

## Commands

### Running the Application

```bash
npm start              # Start the application (runs server.js)
node index.js          # Run directly without time restrictions
node server.js         # Run with time window enforcement (22:55-23:10 UTC)
```

### Notes on Execution

- `server.js` enforces a daily execution window (22:55-23:10 UTC) for automated runs
- `index.js` is the main application entry point without time restrictions
- The application uses `newsnexusdb09` package for database access (local file dependency)

## Architecture

### Core Application Flow

1. **Initialization** (index.js):

   - Load query parameters from Excel spreadsheet
   - Query database to separate never-requested vs previously-requested parameters
   - Sort previously-requested parameters by `dateEndOfRequest` (ascending)
   - Combine into prioritized array: [never-requested, ...oldest-requested-first]

2. **Request Processing Loop**:

   - For each parameter set, calculate `dateEndOfRequest` from database history
   - Verify request is needed (dateEndOfRequest is not today)
   - Make Google News RSS request
   - Parse XML response and store articles in database
   - Respect rate limiting with `MILISECONDS_IN_BETWEEN_REQUESTS` delay
   - Exit after `LIMIT_MAXIMUM_MASTER_INDEX` requests or when all queries completed

3. **Post-Processing**:
   - Optionally trigger semantic scoring via child process (`runSemanticScorer()`)

### Module Organization

- **modules/requestsNewsGoogleRss.js**: Google RSS API requests and article storage

  - `requester()`: Main orchestrator for single request lifecycle
  - `makeGoogleRssRequest()`: Constructs URL, fetches RSS, parses XML
  - `storeNewsApiArticles()`: Saves articles to database (deduplicates by URL)

- **modules/utilitiesReadAndMakeFiles.js**: File I/O operations

  - `getRequestsParameterArrayFromExcelFile()`: Parses Excel spreadsheet to query objects
  - `writeResponseDataFromNewsAggregator()`: Logs API responses to dated directories

- **modules/utilitiesMisc.js**: Database queries and date calculations
  - `createArraysOfParametersNeverRequestedAndRequested()`: Separates queries by request history
  - `checkRequestAndModifyDates()`: Calculates appropriate date ranges for requests
  - `findEndDateToQueryParameters()`: Retrieves last request date for query parameters
  - `runSemanticScorer()`: Spawns child process for semantic analysis

### Database Integration

The application depends on the `newsnexusdb09` package (local file dependency from `../NewsNexusDb09`). Key models used:

- **NewsApiRequest**: Tracks all API requests with query parameters and date ranges
- **Article**: Stores article metadata (title, description, URL, publication date)
- **ArticleContent**: Stores full article text content
- **NewsArticleAggregatorSource**: Configuration for news sources (identified by `NAME_APP`)
- **EntityWhoFoundArticle**: Tracks which source/user discovered each article

See `docs/DATABASE_OVERVIEW.md` for complete schema documentation.

### Google News RSS Query Construction

Queries are built from three parameter types (`andString`, `orString`, `notString`):

- AND terms: Space-separated, joined with `AND`
- OR terms: Space-separated, wrapped in parentheses, joined with `OR`
- NOT terms: Space-separated, prefixed with `-` (minus sign)
- Final query: `[AND] [OR] [NOT]` joined by spaces
- Always includes: `language=en` and `country=us` parameters

Example: If `andString="recall"`, `orString="injury death"`, `notString="politics"`:

```
q=recall AND (injury OR death) -politics&language=en&country=us
```

### Request Prioritization Logic

The application ensures comprehensive coverage while avoiding redundant requests:

1. **Never-requested queries** are processed first (no matching `NewsApiRequest` record)
2. **Previously-requested queries** are sorted by `dateEndOfRequest` ascending (oldest first)
3. Each request advances the `dateEndOfRequest` by `requestWindowInDays` (default: 10 days)
4. Requests are skipped if `dateEndOfRequest` equals today's date
5. Process terminates when `LIMIT_MAXIMUM_MASTER_INDEX` reached or all queries current

### Date Range Management

The application implements intelligent date windowing:

- New queries start from 180 days ago
- Subsequent requests continue from last `dateEndOfRequest`
- Request window is 10 days (configurable via `requestWindowInDays`)
- Automatically caps `dateEndOfRequest` at today's date
- Filters out queries already current (dateEndOfRequest === today)

### Error Handling

- API responses saved to `PATH_TO_API_RESPONSE_JSON_FILES/{YYYYMMDD}/` directory
- Failed requests prefixed with `failed_` in filename
- Rate limit detection terminates process gracefully
- XML parsing errors logged with raw XML preserved
- Duplicate articles filtered by URL before insertion

## Environment Variables

All configuration via `.env` file:

**Application Identity:**

- `NAME_APP`: Application name for logging

**Database Configuration:**

- `PATH_DATABASE`: Directory containing SQLite database
- `NAME_DB`: Database filename

**File Paths:**

- `PATH_TO_API_RESPONSE_JSON_FILES`: Directory for API response logs
- `PATH_AND_FILENAME_FOR_QUERY_SPREADSHEET_AUTOMATED`: Excel file with query parameters
- `PATH_AND_FILENAME_TO_SEMANTIC_SCORER`: Path to semantic scorer script
- `PATH_TO_SEMANTIC_SCORER_DIR`: Directory for semantic scorer resources
- `PATH_TO_SEMANTIC_SCORER_KEYWORDS_EXCEL_FILE`: Keywords file for semantic analysis

**News Source Configuration:**

- `NAME_APP`: Must match `nameOfOrg` in `NewsArticleAggregatorSource` table (e.g., "Google News RSS" or "NewsNexusRequesterGoogleRss03")

**Request Control:**

- `ACTIVATE_API_REQUESTS_TO_OUTSIDE_SOURCES`: Set to "true" to enable actual API requests
- `LIMIT_DAYS_BACK_TO_REQUEST`: Days back to request (currently unused in favor of database-driven approach)
- `LIMIT_MAXIMUM_MASTER_INDEX`: Maximum number of requests per run
- `MILISECONDS_IN_BETWEEN_REQUESTS`: Delay between requests for rate limiting

## Excel Spreadsheet Format

Required columns in `PATH_AND_FILENAME_FOR_QUERY_SPREADSHEET_AUTOMATED`:

- `id`: Unique identifier for query row
- `andString`: Space-separated AND terms (supports quoted phrases)
- `orString`: Space-separated OR terms (supports quoted phrases)
- `notString`: Space-separated NOT terms (supports quoted phrases)
- `startDate`: Initial start date for first request (Excel date format)

Notes:

- No `includeDomains` or `excludeDomains` columns needed (removed in current version)
- Date parsing handles Excel serial date format
- Empty strings allowed for any query parameter field

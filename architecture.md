
---

### **MetaMarket Automation: System Architecture**

#### 1. Overview

This document outlines the system architecture for the MetaMarket Automation project. The system is designed as a modular, event-driven pipeline using **Python**, **PostgreSQL**, **RabbitMQ**, and **Celery**. Its core purpose is to perform a daily, automated collection of industry-specific data for a pre-configured list of companies.

The workflow begins with a scheduled job, managed by **Celery Beat**, that triggers parallel data ingestion tasks. These tasks are intelligently routed to different **worker pools** based on their workload profile (CPU-bound vs. I/O-bound) to ensure optimal performance. All gathered text from various sources for a single company is then aggregated into a "daily brief." This brief is then passed to a final Celery task which uses a Large Language Model (LLM) to produce a single, holistic, and concise news summary. This final, curated item is then stored in a relational database for analysis and consumption.

#### 2. Core Architectural Principles

*   **Modularity:** Each part of the system (crawler, searcher, summarizer) is an independent Celery task. This allows for focused development, easier maintenance, and the ability to update one part without impacting others.
*   **Scalability:** The architecture uses dedicated worker pools for different types of jobs. This allows us to scale the number of scraping workers (often CPU-bound) independently from the AI processing workers (I/O-bound), preventing bottlenecks.
*   **Resilience:** The entire workflow is managed by Celery. A failure in a single scraping task will not halt the system. Celery's built-in retry mechanisms can automatically re-run failed summarization or ingestion tasks, ensuring high reliability.
*   **Workload Isolation:** By routing tasks to different queues (`cpu_bound` for scraping, `io_bound` for LLM calls), we prevent long-running, network-waiting tasks from blocking short, intensive scraping tasks. This avoids "pool starvation" and maximizes throughput.

#### 3. Technology Stack

| Component | Technology / Library | Purpose |
| :--- | :--- | :--- |
| **Programming Language** | Python 3.10+ | Core application logic. |
| **Database** | PostgreSQL | Relational data storage for configuration and curated news. |
| **Task Queue / Broker** | RabbitMQ | Manages Celery task queues and the custom `raw_data_queue`. |
| **Task Scheduling** | Celery Beat | Triggering the daily data ingestion process on a recurring schedule. |
| **Distributed Tasks** | Celery | Managing, routing, and executing all asynchronous workloads. |
| **Website Crawling** | `requests` + `Beautiful Soup` or [apify actor](https://apify.com/apify/website-content-crawler) | Primary method for HTML parsing and content extraction. |
| **Fallback Scraper** | [Firecrawl.dev](https://firecrawl.dev/) | Secondary tool for scraping dynamic (JavaScript) or difficult sites. |
| **Twitter Data** | Official X API / Apify Actor | Dual options for fetching tweets based on budget and reliability needs. |
| **AI-Powered Search** | [Perplexity AI](https://www.perplexity.ai/) / [Exa AI](https://exa.ai/) API | High-relevance, keyword-based news discovery. |
| **AI Processing** | OpenAI / Google Gemini | Summarization, titling, and structuring of aggregated data. |

#### 4. Core Components

*   **Scheduler (Celery Beat):** The system's heartbeat. A Celery service that triggers the main `orchestrate_daily_run` task at a fixed time each day (e.g., 01:00 UTC).

*   **Orchestrator Task (A Celery Task):** The conductor of the pipeline. It reads company configurations from PostgreSQL and dispatches the initial data ingestion tasks (`crawl_website_task`, etc.) for each company.

*   **Celery Worker Pools:** Instead of one generic group of workers, we run specialized pools:
    *   **CPU-Bound Workers:** A pool of workers subscribed exclusively to the `cpu_bound` queue. Their job is to execute fast but potentially intensive tasks like scraping websites, parsing HTML, and fetching data from APIs.
    *   **I/O-Bound Workers:** A separate pool of workers subscribed exclusively to the `io_bound` queue. Their only job is to execute the `summarize_brief_task`. They spend most of their time waiting for a network response from the slow LLM API. This separation prevents them from blocking the scrapers.

*   **Message Broker (RabbitMQ):** Serves two distinct but crucial roles:
    1.  **Celery's Backend:** Manages the `cpu_bound` and `io_bound` queues, delivering task "job tickets" from the orchestrator to the correct worker pool.
    2.  **Data Pipeline:** Manages the custom `raw_data_queue`, which is used to decouple the initial data scraping from the aggregation step.

*   **Aggregator Service:** A dedicated, standalone Python script. Its sole responsibility is to consume the small, individual data snippets from the `raw_data_queue`. It bundles all information belonging to the same company into a single "Daily Brief" JSON object. Once a company's brief is complete, it triggers the final step of the pipeline by dispatching the `summarize_brief_task` to Celery.

*   **Summarization Task (A Celery Task):** The final AI-powered component, `summarize_brief_task`, replaces the standalone "Data Processing Engine". It receives the complete "Daily Brief", sends the aggregated text to an LLM, and saves the final, structured summary directly to the PostgreSQL database.

*   **Database (PostgreSQL):** The final destination for company configurations and the curated news items produced by the `summarize_brief_task`.

#### 5. Celery Implementation Example

This refined implementation uses task routing to direct tasks to the appropriate worker pools, creating a highly efficient system.

```python
# tasks.py - A comprehensive example of the Celery setup

from celery import Celery
from celery.schedules import crontab
import pika # Still needed by ingestion tasks to publish to the custom queue

# Configure Celery to use RabbitMQ as the broker
app = Celery('metamarket', broker='amqp://guest@localhost//')

# --- This is the key to workload isolation ---
# Route tasks to their designated queues based on their workload type.
app.conf.task_routes = {
    'tasks.crawl_website_task': {'queue': 'cpu_bound'},
    'tasks.fetch_tweets_task': {'queue': 'cpu_bound'},
    'tasks.search_internet_task': {'queue': 'cpu_bound'},
    'tasks.summarize_brief_task': {'queue': 'io_bound'}, # The LLM task goes here
}

# --- Task Definitions ---

@app.task
def orchestrate_daily_run():
    # 1. Fetch all company configurations from PostgreSQL
    companies = get_companies_from_db()
    for company in companies:
        # 2. Dispatch ingestion tasks. Celery automatically routes them to the 'cpu_bound' queue.
        crawl_website_task.delay(company.id, company.website_targets)
        # ... other ingestion tasks ...

@app.task
def crawl_website_task(company_id, targets):
    # Logic to crawl websites as described...
    scraped_text = "..."
    # 3. Publish result to our custom RabbitMQ queue for aggregation
    message = { "company_id": company_id, "scraped_text": scraped_text, ... }
    publish_to_rabbitmq('raw_data_queue', message)

@app.task
def crawl_website_task(company_id, targets):
    # Logic to crawl websites as described...
    scraped_text = "..."
    source_url = "..."
    
    # 3. Publish result directly to our custom RabbitMQ queue for aggregation
    message = {
        "company_id": company_id,
        "source_type": "website",
        "source_url": source_url,
        "scraped_text": scraped_text
    }
    publish_to_rabbitmq('raw_data_queue', message)

@app.task
def fetch_tweets_task(company_id, twitter_handle):
    # ... logic to fetch tweets ...
    pass # Publishes to raw_data_queue as well

@app.task
def search_internet_task(company_id, keywords):
    # ... logic to search the web ...
    pass # Publishes to raw_data_queue as well

@app.task(bind=True, max_retries=3, default_retry_delay=60)
def summarize_brief_task(self, daily_brief):
    """
    This is the I/O-bound task that replaces the standalone processing engine.
    It's executed by the I/O-bound worker pool.
    """
    try:
        # 4. Call the slow LLM API
        summary_data = call_llm_api(daily_brief['daily_brief'])
        # 5. Save the final result to the database
        save_to_postgres(company_id=daily_brief['company_id'], data=summary_data)
    except Exception as exc:
        # 6. Use Celery's built-in retry mechanism if the API call fails
        raise self.retry(exc=exc)

# --- Celery Beat Schedule ---
app.conf.beat_schedule = {
    'run-every-day-at-1am': {
        'task': 'tasks.orchestrate_daily_run',
        'schedule': crontab(hour=1, minute=0),
    },
}
```

#### 6. System Data Flow

1.  **Initiation:** **Celery Beat** triggers the `orchestrate_daily_run` task.
2.  **Dispatch:** The Orchestrator fetches company data and dispatches numerous ingestion tasks. Celery routes these tasks to the **`cpu_bound`** queue.
3.  **Ingestion & Publication:**
    *   A **CPU-Bound Worker** picks up an ingestion task (e.g., `crawl_website_task`).
    *   It scrapes the data and publishes the resulting raw text as a small message to the custom **`raw_data_queue`**.
4.  **Aggregation:** The standalone **Aggregator Service** consumes all the small messages from the `raw_data_queue`. It collects all messages for a given company and bundles them into a single, complete "Daily Brief" JSON object.
5.  **Summarization Dispatch:** Once a "Daily Brief" is ready, the Aggregator dispatches the final processing task: `summarize_brief_task.delay(daily_brief)`. Celery automatically routes this task to the **`io_bound`** queue.
6.  **Summarization & Storage:**
    *   An **I/O-Bound Worker**, which has been waiting patiently, picks up the `summarize_brief_task` from the `io_bound` queue.
    *   It makes a slow network call to the LLM API for summarization.
    *   Upon receiving the response, it inserts the final, curated news item directly into the **PostgreSQL** database.

#### 7. RabbitMQ Message & Queue Design

*   **Celery-Managed Queues:** These are used to distribute jobs to the correct worker pools. We don't interact with them directly, but we launch workers to listen to them.
    *   **`cpu_bound`:** A queue for short, intensive scraping and data gathering tasks.
    *   **`io_bound`:** A queue for long-running tasks that wait for network I/O, like LLM API calls.
*   **Custom Data Pipeline Queue:** This is a simple queue we manage for the data flow before final processing.
    *   **`raw_data_queue`**:
        *   **Purpose:** To buffer the small, unstructured pieces of raw data from all sources.
        *   **Producers:** All Celery Data Ingestion tasks (`crawl_website_task`, etc.).
        *   **Consumer:** The `Aggregator Service`.
        *   **Message Content (Example):**
            ```json
            {
              "company_id": 1,
              "source_type": "website",
              "source_url": "https://...",
              "scraped_text": "Tata Steel today announced its financial results..."
            }
            ```

#### 8. Database Schema (PostgreSQL)

*(The database schema remains unchanged, as it is perfectly suited for this improved architecture.)*

```sql
-- Table to store industry verticals
CREATE TABLE industries (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL UNIQUE
);

-- Master table for company configuration
CREATE TABLE companies (
    id SERIAL PRIMARY KEY,
    industry_id INTEGER REFERENCES industries(id),
    name VARCHAR(255) NOT NULL UNIQUE,
    website_targets JSONB,
    search_keywords JSONB,
    twitter_handle VARCHAR(100) UNIQUE
);

-- Table for the final, curated, and summarized output
CREATE TABLE curated_news (
    id SERIAL PRIMARY KEY,
    company_id INTEGER NOT NULL REFERENCES companies(id),
    title VARCHAR(255) NOT NULL,
    summary TEXT NOT NULL,
    primary_source_url VARCHAR(2048) NOT NULL UNIQUE,
    image_url VARCHAR(2048),
    published_date DATE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

Here is an example of how the data for a company like "Tata Steel" would look inside the `companies` table, using the `JSONB` data type for flexibility.

### **Example Data in the `companies` Table**

This SQL `INSERT` statement shows how you would populate the configuration for Tata Steel:

```sql
INSERT INTO companies (industry_id, name, website_targets, search_keywords, twitter_handle)
VALUES (
    1, -- Assuming 'Metals & Mining' has an ID of 1 in the 'industries' table
    'Tata Steel',
    -- This is the JSONB data for website_targets
    '[
        {
            "url": "https://www.tatasteel.com/newsroom/press-releases/",
            "type": "press-release",
            "priority": "high"
        },
        {
            "url": "https://www.tatasteel.com/newsroom/in-the-news/",
            "type": "news-article",
            "priority": "medium"
        }
    ]',
    -- This is the JSONB data for search_keywords
    '["Tata Steel production", "Tata Steel financial results", "decarbonisation", "green steel"]',
    'TataSteelLtd'
);
```

### **How the Row Would Look in the Database**

If you were to query the `companies` table (`SELECT * FROM companies WHERE name = 'Tata Steel';`), the resulting row would look like this:

| id | industry_id | name | website_targets | search_keywords | twitter_handle |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | 1 | Tata Steel | `[{"url": "...", "type": "press-release", "priority": "high"}, {"url": "...", "type": "news-article", "priority": "medium"}]` | `["Tata Steel production", "Tata Steel financial results", "decarbonisation", "green steel"]` | `TataSteelLtd` |
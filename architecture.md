
---

### **MetaMarket Automation: System Architecture**

#### 1. Overview

This document outlines the system architecture for the MetaMarket Automation project. The system is designed as a modular, event-driven pipeline using **Python**, **PostgreSQL**, **RabbitMQ**, and **Celery**. Its core purpose is to perform a daily, automated collection of industry-specific data for a pre-configured list of companies.

The workflow begins with a scheduled job, managed by **Celery Beat**, that triggers parallel data ingestion tasks from company websites, Twitter, and AI-powered search engines. A key feature of this architecture is its efficiency: it first identifies new content published on the current day before committing to a full scrape. All gathered text from various sources for a single company is then aggregated into a "daily brief," which is processed by a Large Language Model (LLM) to produce a single, holistic, and concise news summary. This final, curated item is then stored in a relational database for analysis and consumption.

#### 2. Core Architectural Principles

*   **Modularity:** Each part of the system (crawler, searcher, processor) is an independent component. Celery tasks encapsulate the logic for each data source, allowing for focused development and maintenance.
*   **Scalability:** The use of a RabbitMQ message broker decouples the data ingestion (producers) from the data processing (consumers). Celery workers can be scaled horizontally to handle a large number of data ingestion tasks concurrently.
*   **Resilience:** The event-driven nature, now managed by Celery, ensures that a failure in one scraping task (e.g., one website is down) does not halt the entire system. Celery provides built-in mechanisms for retries and error handling.

#### 3. Technology Stack

| Component | Technology / Library | Purpose |
| :--- | :--- | :--- |
| **Programming Language** | Python 3.10+ | Core application logic. |
| **Database** | PostgreSQL | Relational data storage for configuration and curated news. |
| **Task Queue / Broker** | RabbitMQ | Decoupling services and managing asynchronous tasks with Celery. |
| **Task Scheduling** | Celery Beat | Triggering the daily data ingestion process on a recurring schedule. |
| **Distributed Tasks** | Celery | Managing, distributing, and monitoring the execution of data ingestion tasks. |
| **Website Crawling** | `requests` + `Beautiful Soup` | Primary method for HTML parsing and content extraction. |
| **Fallback Scraper** | [Firecrawl.dev](https://firecrawl.dev/) | Secondary tool for scraping dynamic (JavaScript) or difficult sites. |
| **Twitter Data** | Official X API / Apify Actor | Dual options for fetching tweets based on budget and reliability needs. |
| **AI-Powered Search** | [Perplexity AI](https://www.perplexity.ai/) / [Exa AI](https://exa.ai/) API | High-relevance, keyword-based news discovery. |
| **AI Processing** | OpenAI / Google Gemini | Summarization, titling, and structuring of aggregated data. |

#### 4. Core Components

*   **Scheduler (Celery Beat):** The system's heartbeat. This is a Celery service that triggers tasks on a pre-defined schedule. It replaces the need for a system-level Cron job, providing more flexibility and control within the Python ecosystem. At a fixed time each day (e.g., 01:00 UTC), it dispatches a main "orchestrator" task.

*   **Orchestrator Task (A Celery Task):** This is the main entry point task that acts as the conductor. It reads the company configurations from the PostgreSQL database and then dispatches numerous, parallel data ingestion tasks for each company.

*   **Data Ingestion Workers (Celery Workers):** These are the processes that execute the actual data-gathering logic. Each worker listens for tasks from the RabbitMQ broker and runs them. The ingestion modules are defined as distinct Celery tasks:
    *   **`crawl_website_task`:** This task takes a company's website configuration as an argument.
        1.  It crawls the index pages (e.g., `/press-releases`).
        2.  It parses these pages to find links and their associated publication dates.
        3.  **Crucially, if an article's date is today's date, it proceeds to scrape the full text content from that article's page.**
        4.  It then publishes the **scraped text** directly to the `raw_data_queue` in RabbitMQ.
    *   **`fetch_tweets_task`:** Fetches tweets from specified handles using either the Official X API or an Apify Actor. It publishes any new tweets found to the `raw_data_queue`.
    *   **`search_internet_task`:** Uses an AI-native search engine like Perplexity or Exa to find relevant, third-party news. It constructs queries from company names and keywords, filtering for results from the last 24 hours, and publishes findings to the `raw_data_queue`.

*   **Message Broker (RabbitMQ):** Serves two crucial roles:
    1.  Acts as the **Celery broker**, managing the distribution of tasks from the Orchestrator to the Celery Workers.
    2.  Acts as the data pipeline for our custom services, managing the flow of raw and aggregated data through the `raw_data_queue` and `processing_queue`.

*   **Aggregator Service:** A dedicated, non-Celery consumer that reads individual data snippets (a tweet, a scraped article) from the `raw_data_queue`. It bundles all information belonging to the same company into a single "Daily Brief" object.

*   **Data Processing Engine:** The AI-powered component. It consumes the "Daily Brief" from the `processing_queue`, sends the aggregated text to an LLM, and receives a structured, summarized response.

*   **Database (PostgreSQL):** The final destination. It stores the company configurations and the final, curated news items produced by the Data Processing Engine.

#### 5. Celery Implementation Example

Integrating Celery involves defining tasks with simple decorators. This allows the system to easily manage and distribute the work.

```python
# tasks.py - A simplified example of Celery task definitions

from celery import Celery
import pika # For publishing results to our custom queue

# Configure Celery to use RabbitMQ as the broker
app = Celery('metamarket', broker='amqp://guest@localhost//')

# This is the main task triggered by Celery Beat
@app.task
def orchestrate_daily_run():
    # 1. Fetch all company configurations from PostgreSQL
    companies = get_companies_from_db()
    
    for company in companies:
        # 2. Dispatch ingestion tasks for each company to run in parallel
        crawl_website_task.delay(company.id, company.website_targets)
        fetch_tweets_task.delay(company.id, company.twitter_handle)
        search_internet_task.delay(company.id, company.search_keywords)

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

# Celery Beat Schedule Configuration
app.conf.beat_schedule = {
    'run-every-day-at-1am': {
        'task': 'tasks.orchestrate_daily_run',
        'schedule': crontab(hour=1, minute=0), # Runs daily at 01:00 UTC
    },
}
```

#### 6. System Data Flow

1.  **Initiation:** **Celery Beat** triggers the `orchestrate_daily_run` task at the scheduled time.
2.  **Dispatch:** The **Orchestrator Task** fetches company configurations from PostgreSQL and dispatches `crawl_website_task`, `fetch_tweets_task`, and `search_internet_task` for each company. Celery places these tasks onto its queue in RabbitMQ.
3.  **Ingestion & Publication:**
    *   Available **Celery Workers** pick up the ingestion tasks.
    *   A worker running `crawl_website_task` finds an article published today, scrapes its full text, and publishes a message `{company_id, scraped_text, source_url}` to the **`raw_data_queue`**.
    *   Other workers do the same for Twitter and Internet Search tasks, also publishing their findings to the `raw_data_queue`.
4.  **Aggregation:** The standalone **Aggregator Service** consumes from `raw_data_queue`. It collects all messages for a given `company_id` over a time window. Once complete, it bundles them into a single JSON object.
5.  **Bundling & Forwarding:** The Aggregator publishes the complete "Daily Brief" JSON object to the **`processing_queue`**.
6.  **Summarization:** The **Data Processing Engine** consumes the "Daily Brief" from `processing_queue`. It sends all the combined text to an LLM API for summarization.
7.  **Final Storage:** The engine receives the structured summary from the LLM and inserts it as a new row into the `curated_news` table in the **PostgreSQL** database.

#### 7. RabbitMQ Message & Queue Design

*   **Celery Queues:** Celery will automatically create and manage its own exchanges and queues within RabbitMQ for task distribution. We simply point Celery to the RabbitMQ instance.
*   **Custom Data Queues:** Our custom services use a dedicated exchange and queues for the data pipeline itself.
*   **Exchange:** A single `direct` exchange: `market_automation_exchange`.
*   **Queues & Routing Keys:**
    1.  **`raw_data_queue`**:
        *   **Routing Key:** `raw.data`
        *   **Producers:** All Celery Data Ingestion tasks (`crawl_website_task`, etc.).
        *   **Consumer:** `Aggregator Service`.
        *   **Message Content (Example from Website Crawler):**
            ```json
            {
              "company_id": 1,
              "source_type": "website",
              "source_url": "https://www.tatasteel.com/newsroom/press-releases/tata-steel-q2-results/",
              "scraped_text": "Mumbai, October 4, 2025: Tata Steel today announced its financial results for the second quarter... The company reported a consolidated Profit After Tax (PAT) of ₹X,XXX crore..."
            }
            ```
    2.  **`processing_queue`**:
        *   **Routing Key:** `aggregated.data`
        *   **Producer:** `Aggregator Service`.
        *   **Consumer:** `Data Processing Engine`.
        *   **Message Content (Example "Daily Brief"):**
            ```json
            {
              "company_id": 1,
              "daily_brief": {
                "website_articles": [
                  {"url": "...", "text": "Mumbai, October 4, 2025: Tata Steel today announced..."}
                ],
                "tweets": [
                  {"url": "...", "text": "Our commitment to #GreenSteel continues as we invest in new technologies..."}
                ],
                "external_news": []
              }
            }
            ```

#### 8. Database Schema (PostgreSQL)

*(The database schema remains unchanged as its structure is independent of the task execution framework.)*

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
    -- Using JSONB is highly flexible for storing multiple target URLs and keywords
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
    -- The most representative source URL for the daily summary
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

---

### **MetaMarket Automation: System Architecture**

#### 1. Overview

This document outlines the system architecture for the MetaMarket Automation project. The system is designed as a modular, event-driven pipeline using **Python**, **PostgreSQL**, and **RabbitMQ**. Its core purpose is to perform a daily, automated collection of industry-specific data for a pre-configured list of companies.

The workflow begins with a scheduled job that triggers parallel data ingestion from company websites, Twitter, and AI-powered search engines. A key feature of this architecture is its efficiency: it first identifies new content published on the current day before committing to a full scrape. All gathered text from various sources for a single company is then aggregated into a "daily brief," which is processed by a Large Language Model (LLM) to produce a single, holistic, and concise news summary. This final, curated item is then stored in a relational database for analysis and consumption.

#### 2. Core Architectural Principles

*   **Modularity:** Each part of the system (crawler, searcher, processor) is an independent component. This allows for focused development, easier maintenance, and the ability to update one part without impacting the others.
*   **Scalability:** The use of a RabbitMQ message broker decouples the data ingestion (producers) from the data processing (consumers). This allows the system to handle a large number of sources and scale the processing workers independently as needed.
*   **Resilience:** The event-driven nature ensures that a failure in one scraping task (e.g., one website is down) does not halt the entire system. Failed messages can be re-queued or routed to a dead-letter queue for later inspection.



#### 3. Technology Stack

| Component | Technology / Library | Purpose |
| :--- | :--- | :--- |
| **Programming Language** | Python 3.10+ | Core application logic. |
| **Database** | PostgreSQL | Relational data storage for configuration and curated news. |
| **Message Broker** | RabbitMQ | Decoupling services and managing the asynchronous data flow. |
| **Scheduling** | Cron (Linux) | Triggering the daily data ingestion process. |
| **Website Crawling** | `requests` + `Beautiful Soup` | Primary method for HTML parsing and content extraction. |
| **Fallback Scraper** | [Firecrawl.dev](https://firecrawl.dev/) | Secondary tool for scraping dynamic (JavaScript) or difficult sites. |
| **Twitter Data** | Official X API / Apify Actor | Dual options for fetching tweets based on budget and reliability needs. |
| **AI-Powered Search** | [Perplexity AI](https://www.perplexity.ai/) / [Exa AI](https://exa.ai/) API | High-relevance, keyword-based news discovery. |
| **AI Processing** | OpenAI  / Google Gemini | Summarization, titling, and structuring of aggregated data. |

#### 4. Core Components

*   **Scheduler (Cron Job):** The system's heartbeat. A simple Cron job on a Linux server triggers the main Python script at a fixed time each day (e.g., 01:00 UTC).

*   **Orchestration Service:** The main Python script that acts as the conductor. It reads the company configurations from the database and dispatches initial tasks for each company to the appropriate data ingestion modules.

*   **Data Ingestion Modules:** A set of specialized workers responsible for fetching raw data.
    *   **Website Crawler:** This module is responsible for company websites.
        1.  It first crawls the index pages (e.g., `/press-releases`).
        2.  It parses these pages to find links and their associated publication dates.
        3.  **Crucially, if an article's date is today's date, it immediately proceeds to scrape the full text content from that article's page.**
        4.  It then publishes the **scraped text** to the `raw_data_queue`.
    *   **Twitter Crawler:** Fetches tweets from specified handles. Two primary methods are available:
        *   **Option A: X Official API:** The most reliable and officially supported method. It guarantees high-quality data but comes with API costs and strict rate limits that must be managed. Best for production-critical systems.
        *   **Option B: Apify Actor:** A robust, third-party scraping solution that can be more cost-effective. It is excellent at handling logins and navigating complex UIs, making it a powerful alternative if the official API costs are a concern.
    *   **Internet Searcher:** Uses an AI-native search engine like Perplexity or Exa to find relevant, third-party news. It constructs queries from company names and keywords, filtering for results from the last 24 hours.

*   **Message Broker (RabbitMQ):** The central nervous system of the architecture, managing the flow of data between components through a set of exchanges and queues.

*   **Aggregator Service:** A dedicated consumer that reads individual data snippets (a tweet, a scraped article) from the `raw_data_queue`. It bundles all information belonging to the same company into a single "Daily Brief" object.

*   **Data Processing Engine:** The AI-powered component. It consumes the "Daily Brief" from the `processing_queue`, sends the aggregated text to an LLM, and receives a structured, summarized response.

*   **Database (PostgreSQL):** The final destination. It stores the company configurations and the final, curated news items produced by the Data Processing Engine.



#### 5. System Data Flow

1.  **Initiation:** A **Cron Job** triggers the **Orchestration Service**.
2.  **Dispatch:** The Orchestrator fetches company configurations from PostgreSQL and dispatches tasks for each company to the ingestion modules.
3.  **Ingestion & Publication:**
    *   The **Website Crawler** finds an article published today, scrapes its full text, and publishes a message `{company_id, scraped_text, source_url}` to the **`raw_data_queue`**.
    *   The **Twitter Crawler** finds a new tweet and publishes its content to the same queue.
    *   The **Internet Searcher** finds a relevant news story and publishes its scraped text to the same queue.
4.  **Aggregation:** The **Aggregator Service** consumes from `raw_data_queue`. It collects all messages for a given `company_id` over a time window. Once complete, it bundles them into a single JSON object.
5.  **Bundling & Forwarding:** The Aggregator publishes the complete "Daily Brief" JSON object to the **`processing_queue`**.
6.  **Summarization:** The **Data Processing Engine** consumes the "Daily Brief" from `processing_queue`. It sends all the combined text to an LLM API for summarization.
7.  **Final Storage:** The engine receives the structured summary from the LLM and inserts it as a new row into the `curated_news` table in the **PostgreSQL** database.

#### 6. RabbitMQ Message & Queue Design

*   **Exchange:** A single `direct` exchange: `market_automation_exchange`.
*   **Queues & Routing Keys:**
    1.  **`raw_data_queue`**:
        *   **Routing Key:** `raw.data`
        *   **Producers:** All Data Ingestion Modules.
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

#### 7. Database Schema (PostgreSQL)

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
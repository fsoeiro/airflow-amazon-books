# Amazon Books ETL Pipeline with Airflow & Postgres (Docker-Based)

This is a small learning project to **scrape Amazon books data**, transform it into structured records, and store the results in a **Postgres** database.  
The repository serves as a lightweight template for experimenting with **Airflow + Postgres** in a local Docker environment and for practicing ETL orchestration patterns.

Inspired by the tutorial from *Sunjana in Data*.

---

## Project Goal

Provide a minimal, reproducible Airflow-based ETL example that demonstrates how to:
- Extract web data (Amazon book listings) using Python and BeautifulSoup,
- Transform and deduplicate the scraped results into a tabular format (Pandas DataFrame),
- Temporarily exchange data between tasks using Airflow **XCom**,
- Load the cleaned records into a Postgres table using Airflow hooks/operators.

---

## High-Level Implementation (Overview)

- **Orchestration:** Apache Airflow (DAG defines the workflow and task dependencies).  
- **Extraction:** A Python function performs HTTP requests against Amazon search pages, parses HTML with BeautifulSoup, and builds a list of book records.  
- **Transformation:** Results are converted into a Pandas DataFrame, duplicates are removed (by title), and the records are serialized into XCom.  
- **Loading:** Another Python task pulls the data from XCom and inserts rows into a Postgres `books` table using `PostgresHook`. A `PostgresOperator` ensures the `books` table exists prior to inserts.  
- **Local execution environment:** Docker Compose with Airflow services and a Postgres database (used in development/testing).

---

## DAG Summary

The DAG (`fetch_and_store_amazon_books`) implements three main tasks:

1. **fetch_book_data** (PythonOperator)  
   - Scrapes Amazon search results for book metadata (title, author, price, rating).  
   - Stores the payload in XCom with key `book_data`.

2. **create_table** (PostgresOperator)  
   - Creates the `books` table if it does not exist

3. **insert_book_data** (PythonOperator)
   - Pulls `book_data` from XCom and inserts each record into Postgres using parameterized SQL via `PostgresHook`.

**Task dependency:**  
`fetch_book_data >> create_table >> insert_book_data`

---

## Key Design Decisions & Notes

### **XCom as a light message bus**
The pipeline uses XCom to transfer the scraped payload between tasks. This is convenient for small payloads and local testing; for larger datasets or production use, prefer external storage (S3, object store, or a staging table).

### **Scraping considerations**
The scraper relies on HTML structure and user-agent headers; Amazon’s markup can change and scraping may be subject to rate limits, CAPTCHAs, or legal/terms restrictions—use responsibly and consider official APIs where available.

### **Idempotence & duplication**
The transform step deduplicates by title to reduce redundant inserts, but more robust deduplication (e.g., unique constraints, upserts) is recommended for production.

### **Local dev focus**
The stack is designed for local experimentation with Docker Compose. Production deployments should include secret management, dedicated scheduler/executor, retry/failure strategies, and monitoring.

---

## Tech Stack

- **Apache Airflow** (DAG orchestration)  
- **Docker / Docker Compose** (local environment)  
- **Python** (scraper & operators)  
  - `requests`, `beautifulsoup4`, `pandas`  
- **PostgreSQL** (target storage)  
- **Airflow providers:** `apache-airflow-providers-postgres`

---

## Potential Improvements (Future Work)

- Replace XCom transfer with a staging layer (S3 or a Postgres staging table) for larger data volumes.  
- Add robust retry/backoff and exception handling for network/scraping failures.  
- Implement idempotent upserts (e.g., `INSERT ... ON CONFLICT` or `MERGE`) to avoid duplicates at load time.  
- Add data quality checks (row counts, null thresholds, schema validation) as Airflow tasks.  
- Use secret managers for credentials.  
- Replace naive HTML parsing with resilient extraction (structured API or headless browser if required).

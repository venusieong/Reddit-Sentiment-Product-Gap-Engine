# Reddit Product Gap Engine: Roadmap

## Vision
 To build an automated competitive intelligence system that identifies "Product Gaps"—unfilled customer needs or competitor weaknesses—by transforming unstructured social discourse into high-frequency, actionable data. 

### The Problem:
 * Traditional Feedback is Slow: Surveys take weeks to collect and analyze.
 * Selection Bias: Company-led surveys only ask what the company thinks matters.
 * Competitor Blindspot: Organizations rarely have a real-time, objective view of why users are abandoning their competitors.

### The Solution: "Market Radar"
This engine provides an unfiltered "Radar" for the market. By using AI to extract Unknown Unknowns (issues we didn't even know existed), we allow product teams to react to competitor failures or emerging trends in hours, not months.

-------------------------------------------------------------------------------------------

### Success Metrics (KPIs)
* **Data Freshness:** New Reddit insights must be available in the dashboard within 1 hour of ingestion.
* **Sentiment Precision:** Only NLP results with a confidence score > 0.85 will be used for "Product Gap" flagging.
* **Pipeline Reliability:** 0% data loss during the transfer from PostgreSQL to S3 Bronze.

------------------------------------------------------------------------------------------

### Project Scope
Defining the scope ensures you build a Minimum Viable Product (MVP) that works, rather than a massive system that never gets finished.

### In-Scope (What we are building)
- Batch Ingestion: Hourly extraction of posts and comments from up to 5 specific subreddits (e.g., r/technology, r/skincare).

- Competitor Benchmarking: Tracking up to 3 competitor brands alongside a primary brand for side-by-side comparison.

- NLP Enrichment: Automated sentiment labeling (Positive, Negative, Neutral) and key phrase extraction (Product Gaps) using managed AI.

- Medallion Storage: A structured data lake approach (Bronze/Silver/Gold) on S3.

- Interactive Dashboard: A visual interface showing sentiment trends and top "gap" phrases over time.

### Out-of-Scope (What we are NOT building yet)
- Real-time Streaming: We are using Batch processing (hourly), not sub-second streaming (like Kafka/Flink).

- Multi-language Support: The initial engine will focus exclusively on English-language text.

- Image/Video Analysis: The pipeline will ignore media and focus solely on text-based feedback.

- Automated Response: The engine provides insights but does not automatically post replies back to Reddit.

--------------------------------------------------------------------------------------------
### Tech Stack
| Component | Technology | Rationale |
| :--- | :--- | :--- |
| Orchestration	| Apache Airflow | The industry standard for managing complex task dependencies and retries. |
| Source | APIPRAW (Python) | The most robust and well-documented wrapper for the Reddit API. |
| StagingDB | PostgreSQL | Provides relational integrity and a local "Source of Truth" for raw data. |
| Data Lake | Amazon S3 | Offers infinite, low-cost storage and integrates perfectly with AWS compute. |
| Compute Engine | AWS Glue (Spark) | Serverless PySpark allows for high-power cleaning without managing servers. | 
| NLP / AI | AWS Comprehend | A managed service that provides high-accuracy sentiment without needing a data scientist. | 
| Data Warehouse | Amazon Redshift | Optimized for complex analytical queries and large-scale data aggregation. | 
| Visualization | Amazon QuickSight | Direct integration with Redshift for building low-latency dashboards. |

-------------------------------------------------------------------------------------------
## Implementation Roadmap

### Phase 1: Local Ingestion & Staging (Bronze-Zero)
Goal: Establish a reliable "Source of Truth" for raw data.

##### [ ] Infrastructure: Dockerize Apache Airflow and PostgreSQL.
##### [ ] API Ingestion: Develop Python scripts using PRAW to fetch Reddit submissions.
##### [ ] Data Persistence: Design a PostgreSQL schema with JSONB support for raw payloads.
##### [ ] Orchestration: Build an Airflow DAG to automate hourly ingestion and handle API rate limits.

### Phase 2: Cloud Migration (Bronze)
Goal: Scale storage to the AWS Data Lake.

##### [ ] AWS Setup: Configure S3 buckets (Bronze, Silver, Gold) and IAM security policies.
##### [ ] Incremental Loading: Implement a "Watermark" strategy to only move new records from Postgres to S3.
##### [ ] File Format Optimization: Convert raw data to Parquet during the S3 upload for cost-efficient storage.

### Phase 3: AI Intelligence & Transformation (Silver)
Goal: Convert noise into structured product insights.

##### [ ] Glue ETL: Develop PySpark jobs to clean text (stripping URLs/bots/HTML noise).
##### [ ] NLP Enrichment: Integrate AWS Comprehend for:
* Sentiment Analysis (Positive/Negative/Neutral).
* Key Phrase Extraction (Discovering the "Product Gaps").
##### [ ] Schema Evolution: Store enriched data in the S3 Silver layer.

### Phase 4: Data Warehousing (Gold)
Goal: High-performance analytical serving.

##### [ ] Redshift Setup: Deploy Amazon Redshift (Serverless).
##### [ ] Data Loading: Implement COPY commands to move Silver data into analytical tables.
##### [ ] Modeling: Create SQL views to aggregate metrics like "Sentiment Volatility" and "Competitor Share of Voice."

### Phase 5: Visualization & Delivery
Goal: Empower decision-makers with a "Gap Dashboard."

##### [ ] BI Integration: Connect Amazon QuickSight to Redshift.
##### [ ] Insights: Build a "Market Opportunity Heatmap" based on negative key phrases.
##### [ ] Documentation: Finalize the Technical Design Doc and project post-mortem.
-------------------------------------------------------------------------------------

### Risks & Mitigation
| Risk | Impact | Mitigation Strategy |
| :--- | :--- | :--- |
| Reddit Rate Limiting | High | Use Airflow sensors and watermarking to fetch only incremental data. |
| NLP Cost Spikes | Medium | Batch text processing and truncate long posts before sending to AI. |
| Schema Drift | Low | Use AWS Glue Crawlers to detect and alert on source data changes. |





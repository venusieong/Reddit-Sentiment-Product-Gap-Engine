# Pipeline Design

## Abstract
#### This document outlines the design and implementation of an automated, end-to-end data pipeline for competitive sentiment discovery and "Product Gap" identification. The system leverages a Hybrid Medallion Architecture, orchestrating data movement from a local PostgreSQL staging layer to an AWS S3 data lake via Apache Airflow. By utilizing AWS Glue for serverless PySpark transformations and AWS Comprehend for NLP enrichment, the engine converts unstructured Reddit discourse into structured, engagement-weighted insights. The final output is served through an Amazon Redshift warehouse to power real-time analytical dashboards for market benchmarking.

### System Architecture Diagram
<img src="ArchitectureDiagram.png"/>

Technology Stack
**Orchestration**: Apache Airflow (v2.x) for DAG management and task scheduling.
**Source API**: PRAW (Python) for robust, authenticated Reddit data extraction.
**Staging DB**: PostgreSQL in Docker as the local "Bronze-Zero" relational audit trail.
**Data Lake**: Amazon S3 organized via Medallion layers (Bronze/Silver/Gold) in Parquet.
**Compute**: AWS Glue (PySpark) for serverless, distributed data transformation.
**Intelligence**: AWS Comprehend for managed NLP sentiment and entity extraction.
**Warehouse**: Amazon Redshift (Serverless) for high-performance analytical SQL queries.
**Monitoring**: Amazon CloudWatch for centralized logging and pipeline alerting.
**Visualization**: Amazon QuickSight for building competitive intelligence dashboards.

### Data Lifecycle Deep Dive 

A. Generation (Source System)
- Source: Reddit API via PRAW.
- Characteristics: High-velocity, semi-structured (JSON), and outside of direct engineering control.
- Constraint Handling: The pipeline must adhere to Reddit's Rate Limiting (typically 60-100 requests per minute). 
- The logic implements "Exponential Backoff" to retry requests if throttled.

B. Ingestion Strategy (Bronze-Zero Staging)
- This stage focuses on moving data from the API into our "Bronze-Zero" (Postgres) layer safely.
- Pattern: Incremental Batch Ingestion.
- The High Watermark Strategy: To avoid redundant processing and cost, we use a Watermark column (created_utc).
- Logic: Each run, Airflow queries the maximum created_utc from the Postgres staging table. The next API call specifically requests only posts with a timestamp greater than this value.
- Storage: Data is stored in a JSONB column in PostgreSQL to ensure no data is lost if the Reddit API schema changes ("Schema-on-Read").

C. Transformation (Medallion Flow)
- Data is refined as it moves through the S3 buckets.
- Bronze (Raw): 1:1 copy of the Postgres data, stored as Parquet for cost-effective S3 storage.
- Silver (Enriched): * Text Cleaning: Removal of URLs, bot signatures, and special characters.
- NLP Call: Sentiment and Entity data from AWS Comprehend are appended as new columns.
- Deduplication: Using post_id to ensure absolute uniqueness before moving to Gold.
- Gold (Aggregated): Data is grouped by Brand and Day to create high-speed analytical tables (e.g., daily_sentiment_summary).

D. Serving (Warehouse & BI)
- Pattern: ELT (Extract, Load, Transform).
- Loading: The S3 Gold layer is loaded into Amazon Redshift Serverless using the COPY command, which is optimized for high-volume S3 transfers.
- Data Access: End-users interact with the data via Amazon QuickSight, which queries Redshift directly for near-instant visualization.

**Security**: All API credentials and AWS keys are managed via Environment Variables or AWS Secrets Manager, never hardcoded.
**Observability**: Airflow provides a "Retry" logic (3 attempts per task). If the final attempt fails, an alert is sent via the logging system.

### Primary Source API
**Provider**: Reddit Inc.
**Interface**: Reddit OAuth2 API
**Base URL**: https://oauth.reddit.com
**Access Method**: Python Reddit API Wrapper (PRAW)

### Tables from API
- Link:	The original post in a subreddit (contains the title and main text).
- Comment: User replies within a thread (contains deep-dive complaints and feedback).
- Subreddit: Metadata about the community (subscriber count, rules).

### Schema & Table Definitions
**A.** Operational Tables (PostgreSQL Staging)
These tables live in your local PostgreSQL instance and control the logic of the engine.

- Table: pipeline_config
Defines which brands and competitors are currently being monitored.
| Column | Data Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| config_id | SERIAL | PRIMARY KEY | Unique ID for the configuration version. |
| primary_brand | VARCHAR(50) | NOT NULL | The company's own brand name. |
| competitors | TEXT[] | NOT NULL | Array of competitor brands (e.g., ['Xbox', 'Nintendo']). |
| target_subreddits| TEXT[] | NOT NULL | Subreddits to crawl (e.g., ['r/gaming']). |
| is_active | BOOLEAN | DEFAULT TRUE | Toggle for the Airflow controller. |
| updated_at | TIMESTAMP | DEFAULT NOW() | Tracks when the competitor set was last changed. |

- Table: ingestion_watermarks
Tracks the last post processed for each brand to ensure incremental loading.
| Column | Data Type | Description |
| :--- | :--- | :--- |
| brand_name | VARCHAR(50) | The specific brand being tracked. |
| last_utc | BIGINT | The created_utc of the newest post ingested. |
| last_fullname | VARCHAR(25) | The name (fullname) of the newest post ingested. |


**B.** Data Tables (Bronze-Zero / Staging)
- These tables store the raw fields extracted directly from the Reddit API objects.Table: raw_reddit_data
| Column | Reddit API Field | Data Type | Rationale |
| :--- | :--- | :--- | :--- |
| fullname | name | VARCHAR(25) | PK: Unique identifier (e.g., t3_15bfi0). |
| parent_id | parent_id | VARCHAR(25) | Link to the submission if the record is a comment. |
| title | title | TEXT | Primary subject (Null for comments). |
| body | selftext / body | TEXT | The main text content for NLP analysis. |
| upvotes | ups |INTEGER | Used to weight the importance of the feedback. |
| comment_count | num_comments | INTEGER | Engagement metric (Submissions only). | 
| created_at | created_ut | BIGINT | Unix timestamp used for Watermarking. | 
| subreddit | subreddit | VARCHAR(50) | Source community for audience context. |
| raw_payload | - | JSONB | Immutable Truth: Full API response for future-proofing. |

**C.** Enriched Tables (Silver / AWS Data Lake)
- Stored in S3 as Parquet, processed via AWS Glue + Comprehend.
| Column | Data Type | Transformation Logic |
| :--- | :--- | :--- |
| cleaned_text | TEXT | Regex: HTML decoding (e.g., &amp; $\rightarrow$ &) and URL removal. |
| sentiment | VARCHAR(12) | AWS Comprehend: POSITIVE, NEGATIVE, NEUTRAL, MIXED. |
| sentiment_score | DECIMAL | Confidence level of the AI sentiment prediction. |
| gap_entities | ARRAY(TEXT) | AWS Comprehend: Nouns identifying product features (e.g., "battery"). |
| config_version | INT | FK: Links the record to the pipeline_config active at time of ingestion. |

**D.** Analytical Tables (Gold / Redshift)
- Table: fact_product_gaps
| Column | Description |
| :--- | :--- |
| brand_key | Normalized name of the brand (Target or Competitor). |
| obs_date | Date part of the timestamp for time-series charts. | 
| is_gap_signal | TRUE if sentiment is NEGATIVE and upvotes > 10. | 
| feature_topic | The primary "Gap Entity" identified in the text. |



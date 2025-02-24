# SemanticSchemaLinker

**Description:**  
SemanticSchemaLinker is an end-to-end solution that enriches a raw, partitioned PostgreSQL schema with semantic metadata—all processed within PostgreSQL. It supports NL2SQL by linking tables, columns, unique categorical values, relationships, and few-shot query examples using descriptive text and embeddings. The custom VectorStore module is inspired and rebranched from [pgvectorscale-rag-solution](https://github.com/daveebbelaar/pgvectorscale-rag-solution) and uses a local FlashRank reranker so that all processing stays inside PostgreSQL.

## Overview

- **Schema Extraction:**  
  Extract the database structure from parent tables (ignoring partitions) using the `information_schema`.

- **Text Normalization:**  
  For all textual fields (table names, column names, unique values), apply a normalization pipeline:
  ```
  normalize_text(text) = lemmatize_text(remove_stopwords_text(clean_text(text)))
  ```
  (Only applied to string fields; numeric fields are left unchanged.)

- **Description & Embedding Generation:**  
  Construct descriptive prompts for each table and column—including raw DDL for tables—and use Azure OpenAI GPT-4o-Mini to generate descriptions. Then, generate embeddings (1536 dimensions) from the normalized text via the custom VectorStore.

- **Unique Value Extraction:**  
  For categorical columns, if COUNT(DISTINCT) is below a configurable threshold (e.g., 50), extract and normalize those values, then generate their embeddings. These are stored with the country code.

- **Relationship Extraction:**  
  Query system catalogs (e.g., pg_constraint) to extract foreign key relationships. Generate descriptions and embeddings to document join relationships.

- **Few-Shot Example Storage:**  
  Store example NL-to-SQL pairs with associated metadata and embeddings. This table is used to retrieve contextually similar few-shot examples during query generation.

- **Metadata Storage:**  
  Populate the following auxiliary tables:
  - **SCHEMA_TABLES:** Fields: table_name, country_iso3, description, ddl_definition, embedding.
  - **SCHEMA_COLUMNS:** Fields: table_name, country_iso3, column_name, column_type, description, embedding.
  - **SCHEMA_VALUES:** Fields: table_name, country_iso3, column_name, unique_value, embedding.
  - **SCHEMA_RELATIONSHIPS:** Fields: source_table, source_column, target_table, target_column, relationship_type, description, embedding.
  - **SCHEMA_FEWSHOTS:** Fields: question, example_query, table_names, description, embedding.

- **Avoiding Redundant Processing:**  
  Before generating metadata for any element, check whether metadata already exists to avoid duplicate LLM calls.

- **NL2SQL Integration:**  
  When a natural language query is received, generate its embedding and compare it with stored embeddings to identify the most relevant tables, columns, values, and few-shot examples for query generation.

## Technologies & Dependencies

- **PostgreSQL** with the pgvector extension and Timescale Vector for vector storage and search.
- **Azure OpenAI GPT-4o-Mini** for generating descriptive text and embeddings (model: text-embedding-3-small, 1536 dimensions).
- **Python 3.x** with SQLAlchemy, Pandas, Loguru, and psycopg.
- **Custom VectorStore Module:** Inspired by [pgvectorscale-rag-solution](https://github.com/daveebbelaar/pgvectorscale-rag-solution), it integrates Azure OpenAI embeddings and supports local reranking using FlashRank (via langchain_community.document_compressors.flashrank_rerank and flashrank).
- Text processing functions: `clean_text`, `remove_stopwords_text`, and `lemmatize_text`.

## Repository Structure

- **/data_ingestion/**: Scripts for schema extraction and metadata generation.
- **/modules/config.py**: Configuration settings (database connection strings, API keys, etc.).
- **/modules/text_processing.py**: Contains text normalization functions.
- **/modules/vector_store.py**: Encapsulates Azure OpenAI embeddings, vector search, and local reranking.
- **README.md**: Project documentation.

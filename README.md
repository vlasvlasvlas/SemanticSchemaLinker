# SemanticSchemaLinker

**Description:**  
SemanticSchemaLinker is an end-to-end solution for enriching a raw, partitioned PostgreSQL schema with semantic metadata—all within PostgreSQL. It supports NL2SQL applications by linking tables, columns, categorical values, relationships (joins), and few-shot query examples using descriptive text and semantic embeddings. The custom VectorStore module is inspired and rebranched from [pgvectorscale-rag-solution](https://github.com/daveebbelaar/pgvectorscale-rag-solution) and incorporates a local reranker so that all vector operations and searches remain entirely inside PostgreSQL.

## Overview

SemanticSchemaLinker automatically extracts your database schema, normalizes text fields (using custom functions like `clean_text`, `remove_stopwords_text`, and `lemmatize_text`), and generates rich descriptions using an LLM (via Azure OpenAI). It then creates embeddings (1536 dimensions) for:
- **Tables (SCHEMA_TABLES):** Stores high-level metadata, including the raw DDL to show partitioning details.
- **Columns (SCHEMA_COLUMNS):** Provides detailed metadata for each column, including data type and descriptive embeddings.
- **Values (SCHEMA_VALUES):** For categorical columns with low distinct counts (e.g., less than 50), stores unique values and their embeddings, distinguished by country.
- **Relationships (SCHEMA_RELATIONSHIPS):** Captures foreign key and join information with descriptive metadata and embeddings.
- **Few-Shots (SCHEMA_FEWSHOTS):** Contains example NL-to-SQL pairs with embeddings for context-aware query generation.

## Process Flow

1. **Schema Extraction:**  
   - Connect to PostgreSQL and retrieve the list of parent tables (ignoring individual partitions) and their columns via the `information_schema`.

2. **Normalization:**  
   - Apply text normalization (cleaning, stopword removal, lemmatization) on all string fields (table names, column names, and unique categorical values) to ensure consistency, while leaving numeric fields untouched.

3. **Description & Embedding Generation:**  
   - For each table and column, generate descriptive prompts (including raw DDL for tables) and pass these to the LLM to produce rich descriptions.
   - Generate semantic embeddings using Azure OpenAI (model: text-embedding-3-small, 1536 dimensions) via the custom VectorStore module.

4. **Unique Value Extraction:**  
   - For each categorical column, if the COUNT(DISTINCT) is below a configurable threshold (e.g., 50), extract and normalize those unique values and generate their embeddings.

5. **Relationship Extraction:**  
   - Query PostgreSQL’s system catalogs (e.g., `pg_constraint`) to detect foreign key relationships, then generate descriptive metadata and embeddings for the joins.

6. **Few-Shot Example Storage:**  
   - Collect NL-to-SQL example pairs, generate descriptive embeddings, and store them to assist the NL2SQL engine with contextual guidance.

7. **Storage:**  
   - Insert all metadata into five auxiliary tables:  
     - **SCHEMA_TABLES:** table_name, country_iso3, description, ddl_definition, embedding  
     - **SCHEMA_COLUMNS:** table_name, country_iso3, column_name, column_type, description, embedding  
     - **SCHEMA_VALUES:** table_name, country_iso3, column_name, unique_value, embedding  
     - **SCHEMA_RELATIONSHIPS:** source_table, source_column, target_table, target_column, relationship_type, description, embedding  
     - **SCHEMA_FEWSHOTS:** question, example_query, table_names, description, embedding  
   - Before processing any element, the system checks if metadata already exists to avoid redundant processing.

8. **Usage in NL2SQL:**  
   - Upon receiving a natural language query, the system generates its embedding and compares it against the stored metadata embeddings to determine the most relevant schema components and few-shot examples for constructing the SQL query.

## Technologies & Dependencies

- **PostgreSQL** with the **pgvector** extension and **Timescale Vector** for efficient vector storage and search.
- **Azure OpenAI Embeddings** (model: text-embedding-3-small, 1536 dimensions).
- **Python 3.x** with libraries such as SQLAlchemy, Pandas, and Loguru.
- **Custom VectorStore Module:** Inspired and rebranched from [pgvectorscale-rag-solution](https://github.com/daveebbelaar/pgvectorscale-rag-solution), it integrates Azure OpenAI embeddings and supports local reranking using FlashRank (via `langchain_community.document_compressors.flashrank_rerank` and `flashrank`), ensuring that all processing stays inside PostgreSQL.
- Additional libraries: `psycopg`, `langchain_openai`, etc.

## Repository Structure

- **/data_ingestion/**: Scripts for schema extraction, metadata generation, and embedding insertion.
- **/modules/config.py**: Configuration settings (database connection strings, API keys, etc.).
- **/modules/text_processing.py**: Contains `clean_text`, `remove_stopwords_text`, `lemmatize_text`.
- **/modules/vector_store.py**: Encapsulates Azure OpenAI embeddings, vector search with Timescale Vector, and local reranking.
- **README.md**: Documentation of the project.

## Conclusion

SemanticSchemaLinker is a comprehensive, PostgreSQL-integrated solution that leverages semantic embeddings to bridge raw database schemas with NL2SQL applications. By extracting, normalizing, and enriching schema metadata—and by including relationship and few-shot context—it significantly enhances query generation accuracy. The solution, built with custom vector store logic and a local reranker inspired by pgvectorscale-rag-solution, ensures that all processing remains within PostgreSQL, making it scalable, efficient, and cost-effective.

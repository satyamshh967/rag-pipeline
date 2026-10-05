AI Knowledge Pipeline
A backend-focused Retrieval-Augmented Generation (RAG) system that ingests knowledge from web pages, processes and stores that knowledge, retrieves relevant information using semantic search, and generates grounded answers through an LLM.
Project Status
Current stage: Core RAG pipeline implemented and being prepared for further production-oriented development.
Currently implemented
Web URL ingestion
Webpage content extraction
Document validation with Pydantic
Overlapping text chunking
Sentence Transformer embeddings
ChromaDB vector storage
Semantic similarity retrieval
Relevance-score filtering
RAG context construction
OpenRouter LLM integration
Source attribution
PostgreSQL document persistence
Document create/list/get/update/delete operations
SQLAlchemy repository layer
Request logging
Request IDs
Global error handling
Docker and Docker Compose
PostgreSQL health checks
Automated tests
Retrieval evaluation pipeline
Environment-based configuration
The project is intentionally still evolving. The future plan is to turn this core RAG backend into a more complete production-oriented AI knowledge platform.
---
Why I Built This
A language model alone does not automatically know the information contained in a private or user-provided knowledge base.
Retrieval-Augmented Generation solves this by separating the problem into two stages:
Retrieve relevant information.
Generate an answer using the retrieved information.
This project was built to understand that complete pipeline from the backend side rather than simply calling an LLM API.
The core flow is:
```text
User Question
      |
      v
Query Embedding
      |
      v
Semantic Retrieval
      |
      v
Relevant Chunks
      |
      v
Context Construction
      |
      v
LLM
      |
      v
Grounded Answer + Sources
```
---
Architecture
```text
                         Client
                           |
                           v
                    +-------------+
                    |   FastAPI   |
                    |     API     |
                    +------+------+
                           |
             +-------------+-------------+
             |                           |
             v                           v
      Document Operations           Query / RAG
             |                           |
             v                           v
     DocumentService             RetrievalService
             |                           |
       +-----+------+                    |
       |            |                    |
       v            v                    |
 Web Ingestion   PostgreSQL              |
       |            |                    |
       v            |                    |
    Chunking        |                    |
       |            |                    |
       v            |                    |
  Embeddings       |                    |
       |            |                    |
       v            |                    |
    ChromaDB <------+--------------------+
       |
       v
 Retrieved Chunks
       |
       v
 Context Builder
       |
       v
   RAGService
       |
       v
   LLMService
       |
       v
 OpenRouter LLM
       |
       v
 Answer + Sources
```
---
Core Pipeline
1. Web Ingestion
The system accepts a URL and title through:
```http
POST /documents
```
Example:
```json
{
  "url": "https://example.com/article",
  "title": "Example Article"
}
```
`DocumentService` coordinates ingestion.
The service:
Checks whether the source already exists.
Downloads the webpage.
Extracts its content.
Creates a `Document`.
Chunks the document.
Generates embeddings.
Stores chunks and embeddings in ChromaDB.
Persists the document in PostgreSQL.
Relevant modules:
```text
app/ingestion.py
app/chunking.py
app/embeddings.py
app/document_service.py
```
---
2. Document Model
The document model contains:
```text
id
title
source
content
created_at
metadata
```
UUIDs are used for document IDs.
Pydantic provides structured request and model validation.
The document timestamp is generated using a timezone-aware UTC timestamp.
---
3. Chunking
Large documents are divided into smaller overlapping chunks.
Current configuration:
```text
Chunk size: 500 words
Overlap:     50 words
```
The basic process is:
```text
Document
   |
   v
Split into words
   |
   v
500-word chunk
   |
   +---- 50-word overlap ----+
                             |
                             v
                        Next chunk
```
The overlap preserves some context between neighboring chunks.
Each chunk tracks:
```text
id
document_id
content
position
metadata
```
---
4. Embeddings
The project uses:
```text
sentence-transformers/all-MiniLM-L6-v2
```
through the `sentence-transformers` library.
The model is distributed through Hugging Face.
The model converts text into numerical vectors representing semantic meaning.
```text
"What is machine learning?"
             |
             v
      Embedding Model
             |
             v
     [0.12, -0.43, ...]
```
The model name is configured through `app/config.py`, rather than being hardcoded inside the embedding service.
---
5. Vector Storage
Embeddings and chunks are stored in ChromaDB.
ChromaDB is responsible for semantic vector search.
Stored metadata includes:
```text
document_id
title
source
position
```
The vector-store abstraction is implemented in:
```text
app/vector_store.py
```
This keeps vector database operations separate from API and business logic.
---
6. PostgreSQL Persistence
PostgreSQL stores application-level document information.
The SQLAlchemy database model is:
```text
DocumentRecord
```
and the database table is:
```text
documents
```
Current fields:
```text
id
title
source
content
created_at
```
The database stack is:
```text
PostgreSQL 16
SQLAlchemy
psycopg
```
The project intentionally separates relational document storage from vector storage:
```text
PostgreSQL
    |
    +-- Document metadata
    +-- Document content
    +-- Source
    +-- Creation timestamp

ChromaDB
    |
    +-- Chunk embeddings
    +-- Chunk content
    +-- Retrieval metadata
```
This also provides a foundation for future users, ownership, collections, permissions, and relationships.
---
7. Repository Layer
Document persistence is abstracted through:
```text
DocumentRepository
```
The repository exposes:
```text
add()
get()
get_all()
get_by_source()
update()
delete()
```
The repository uses SQLAlchemy sessions rather than exposing database queries directly to the API.
Architecture:
```text
FastAPI
   |
   v
DocumentService
   |
   v
DocumentRepository
   |
   v
SQLAlchemy
   |
   v
PostgreSQL
```
---
8. Document Lifecycle
Ingestion
```text
URL
 |
 v
Fetch webpage
 |
 v
Document
 |
 v
Chunk
 |
 v
Embedding
 |
 v
ChromaDB
 |
 v
PostgreSQL
```
Update
When a document is updated:
```text
Existing document
       |
       v
Fetch latest webpage
       |
       v
Re-create chunks
       |
       v
Generate new embeddings
       |
       v
Delete old vectors
       |
       v
Store new vectors
       |
       v
Update document record
```
Delete
```text
Document
   |
   +----> PostgreSQL record removed
   |
   +----> ChromaDB vectors removed
```
Deleting both representations prevents stale vectors from remaining in the vector store.
---
9. Retrieval
`RetrievalService` handles semantic retrieval.
The query flow is:
```text
Question
   |
   v
Embedding Model
   |
   v
Query Vector
   |
   v
ChromaDB similarity search
   |
   v
Candidate chunks
```
The API supports:
```text
top_k
min_score
```
`top_k` is constrained to:
```text
1 <= top_k <= 10
```
Retrieved chunks contain:
```text
content
score
document_id
position
title
source
```
The current relevance score is derived from vector-store distance:
```text
score = 1 / (1 + distance)
```
Chunks below `min_score` are excluded.
---
10. RAG Context Construction
`RAGService` coordinates retrieval and generation.
```text
Question
   |
   v
RetrievalService
   |
   v
Retrieved chunks
   |
   v
Context construction
   |
   v
LLMService
```
Retrieved chunks are combined into a context string and supplied to the LLM together with the original question.
---
11. LLM Integration
The project uses OpenRouter through the OpenAI-compatible API client.
Current model configuration:
```text
openrouter/free
```
The LLM receives:
```text
KNOWLEDGE CONTEXT

+

QUESTION
```
The system prompt instructs the model to:
Use only the provided knowledge context.
Avoid inventing facts.
Avoid unsupported assumptions.
State when the supplied knowledge is insufficient.
The purpose is to keep generation grounded in retrieved information.
---
12. Source Attribution
The `/query` endpoint returns:
```text
answer
sources
```
Each source contains:
```text
document_id
title
source
chunk_position
score
```
Example:
```json
{
  "answer": "Machine learning is ...",
  "sources": [
    {
      "document_id": "document-id",
      "title": "Machine Learning",
      "source": "https://example.com",
      "chunk_position": 8,
      "score": 0.58
    }
  ]
}
```
The API therefore exposes the knowledge used during retrieval rather than returning only generated text.
---
API Reference
GET `/`
Returns basic API status.
GET `/health`
Returns:
```json
{
  "status": "healthy"
}
```
POST `/documents`
Ingests a webpage.
Request:
```json
{
  "url": "https://example.com/article",
  "title": "Example Article"
}
```
Response:
```json
{
  "document_id": "...",
  "title": "Example Article",
  "chunks_created": 20
}
```
GET `/documents`
Returns stored document summaries.
GET `/documents/{document_id}`
Returns the complete document including content and metadata.
PUT `/documents/{document_id}`
Re-fetches, re-chunks, re-embeds, and replaces vectors for an existing document.
DELETE `/documents/{document_id}`
Deletes the document and its associated vectors.
POST `/query`
Runs the complete RAG pipeline.
Request:
```json
{
  "question": "What is machine learning?",
  "top_k": 3,
  "min_score": 0.0
}
```
Response:
```json
{
  "answer": "...",
  "sources": [
    {
      "document_id": "...",
      "title": "...",
      "source": "...",
      "chunk_position": 2,
      "score": 0.58
    }
  ]
}
```
---
Validation and Error Handling
The API uses Pydantic models for request validation.
Examples:
```text
top_k
1 <= top_k <= 10

min_score
0.0 <= min_score <= 1.0
```
The API rejects:
Empty questions
Empty document titles
Invalid UUIDs
Missing required request fields
Invalid values outside configured ranges
Expected errors use appropriate HTTP status codes:
```text
400 → Invalid request
404 → Document not found
500 → Unexpected server error
```
A global exception handler prevents raw internal errors from being exposed to clients.
---
Logging and Observability
The API includes HTTP request logging middleware.
Every request receives a UUID-based request ID.
The logs record:
Request ID
HTTP method
Path
Status code
Request duration
Document operations
Query operations
Retrieved chunk count
Exceptions
Responses include:
```text
X-Request-ID
```
This creates a basic request-tracing mechanism for debugging.
---
Automated Testing
The project contains a Pytest test suite covering the current API and service behavior.
The tests cover areas including:
API endpoints
Validation
Document lifecycle
Retrieval
Service behavior
Query flow
The repository also contains an evaluation pipeline:
```text
evaluation/evaluate.py
evaluation/questions.json
```
The evaluation measures retrieval behavior against predefined questions and expected sources.
---
Evaluation
The current evaluation dataset contains questions about:
```text
Machine Learning
Artificial Intelligence
Deep Learning
```
The evaluation records:
```text
Expected source
Retrieved sources
Retrieval scores
Best retrieval score
Source hit
Keyword score
Matched keywords
```
Current baseline:
```text
Average keyword score:     0.89
Retrieval source accuracy: 0.67
Average retrieval score:   0.562
Source hits:               2/3
```
This baseline provides a measurable reference for future retrieval improvements.
---
Docker
The project is containerized using Docker Compose.
Services:
```text
docker-compose
      |
      +----------------+
      |                |
      v                v
    API              PostgreSQL
  FastAPI              DB
      |
      +---- ChromaDB
      |
      +---- Embedding Model
      |
      +---- OpenRouter
```
The API is exposed on:
```text
http://localhost:8000
```
PostgreSQL runs inside the Compose network.
Database persistence uses a Docker volume.
The API waits for PostgreSQL health before starting through the Compose service dependency configuration.
---
Configuration
Configuration is handled with `pydantic-settings`.
The application reads environment variables from `.env`.
Important configuration:
```text
OPENROUTER_API_KEY
DATABASE_URL
```
The embedding model and vector-store path have application defaults.
Example:
```env
OPENROUTER_API_KEY=your_openrouter_api_key
DATABASE_URL=postgresql+psycopg://postgres:postgres@db:5432/knowledge
```
The real `.env` file must never be committed.
Use `.env.example` as the template.
---
Project Structure
```text
rag-pipeline/
│
├── app/
│   ├── api.py
│   ├── config.py
│   ├── database.py
│   ├── database_models.py
│   ├── models.py
│   │
│   ├── ingestion.py
│   ├── chunking.py
│   ├── embeddings.py
│   │
│   ├── vector_store.py
│   ├── retrieval.py
│   ├── context.py
│   ├── rag.py
│   ├── llm.py
│   │
│   ├── document_repository.py
│   └── document_service.py
│
├── evaluation/
│   ├── evaluate.py
│   └── questions.json
│
├── tests/
│
├── data/
│
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
├── .env.example
├── .gitignore
└── README.md
```
---
Technology Stack
Technology	Purpose
Python 3.12	Backend
FastAPI	REST API
Pydantic	Validation and models
Pydantic Settings	Configuration
SQLAlchemy	ORM
Psycopg	PostgreSQL driver
PostgreSQL 16	Document persistence
ChromaDB	Vector database
Sentence Transformers	Embeddings
Hugging Face	Model distribution
OpenRouter	LLM access
Docker	Containerization
Docker Compose	Service orchestration
Pytest	Testing
Git/GitHub	Version control
---
Running Locally
Requirements
Docker Desktop
WSL2 on Windows
Git
OpenRouter API key
Clone
```bash
git clone https://github.com/satyamsh967/rag-pipeline.git
cd rag-pipeline
```
Environment
Create `.env`:
```env
OPENROUTER_API_KEY=your_openrouter_api_key
DATABASE_URL=postgresql+psycopg://postgres:postgres@db:5432/knowledge
```
Start
```bash
docker compose up -d --build
```
API:
```text
http://localhost:8000
```
Swagger:
```text
http://localhost:8000/docs
```
Stop:
```bash
docker compose down
```
---
Example End-to-End Workflow
```text
1. Start Docker Compose
        |
        v
2. POST /documents
        |
        v
3. Webpage fetched
        |
        v
4. Document created
        |
        v
5. Document chunked
        |
        v
6. Embeddings generated
        |
        v
7. PostgreSQL stores document
        |
        v
8. ChromaDB stores chunks/vectors
        |
        v
9. POST /query
        |
        v
10. Query embedded
        |
        v
11. ChromaDB retrieves relevant chunks
        |
        v
12. Context constructed
        |
        v
13. OpenRouter generates answer
        |
        v
14. Answer + sources returned
```
---
Current Limitations
The current implementation intentionally focuses on the core RAG architecture.
Known limitations:
Basic webpage extraction
Fixed-size word-based chunking
Semantic retrieval without reranking
Small evaluation dataset
No authentication
No user accounts
No document ownership
No background ingestion workers
No streaming LLM responses
No rate limiting
No production migration framework
Limited metrics and observability
No production deployment
No frontend
These limitations define the next development phases.
---
Future Development Plan
The long-term goal is to evolve this core RAG backend into a complete personal/team knowledge platform.
Phase 1 — Core RAG
Status: Completed
[x] FastAPI REST API
[x] Web ingestion
[x] Document validation
[x] Document chunking
[x] Sentence Transformer embeddings
[x] ChromaDB vector storage
[x] Semantic retrieval
[x] Relevance scoring
[x] RAG context construction
[x] OpenRouter integration
[x] Source attribution
[x] PostgreSQL persistence
[x] Document CRUD
[x] Docker
[x] Docker Compose
[x] Request logging
[x] Error handling
[x] Configuration management
[x] Automated tests
[x] Retrieval evaluation
---
Phase 2 — Authentication and Multi-User Knowledge
Planned
Introduce users and ownership.
Planned features:
JWT authentication
User registration
Login
Password hashing
Access tokens
User-specific documents
Ownership-based authorization
Protected endpoints
User isolation
Target relationship:
```text
User
 |
 +---- Documents
 |
 +---- Collections
 |
 +---- Queries
```
Database direction:
```text
users
  |
  +---- documents
           |
           +---- chunks/vectors
```
---
Phase 3 — Improved Retrieval
Planned
The current system uses vector similarity as its main retrieval mechanism.
Future improvements:
Metadata filtering
Search only within:
A document
A collection
A source
A user's knowledge base
Hybrid search
Combine:
```text
Semantic Search
      +
Keyword Search
```
Reranking
Future retrieval architecture:
```text
Query
  |
  v
Vector Search
  |
  v
Top 20 candidates
  |
  v
Reranker
  |
  v
Top 5 relevant chunks
  |
  v
LLM
```
Retrieval improvements will be evaluated against larger datasets rather than relying only on subjective answer quality.
---
Phase 4 — Better Document Processing
Planned
Expand ingestion beyond webpages.
Potential support:
PDF
Markdown
Plain text
DOCX
File uploads
Multiple URLs
Sitemap ingestion
Potential chunking strategies:
Sentence-aware chunking
Paragraph-aware chunking
Recursive chunking
Token-based chunking
Structure-aware chunking
---
Phase 5 — Background Processing
Planned
Large documents should not block API requests.
Future architecture:
```text
Client
  |
  v
FastAPI
  |
  v
Task Queue
  |
  +------------------+
  |                  |
  v                  v
Ingestion Worker   Embedding Worker
  |                  |
  +--------+---------+
           |
           v
      Vector Store
```
Potential technologies:
```text
Redis
Celery / RQ
Background workers
```
The API could return a job ID while processing continues asynchronously.
---
Phase 6 — Advanced RAG
Planned
The retrieval and generation pipeline can evolve into:
```text
Query
 |
 v
Query preprocessing
 |
 v
Hybrid retrieval
 |
 v
Reranking
 |
 v
Context compression
 |
 v
Context assembly
 |
 v
LLM
 |
 v
Answer + citations
```
Potential features:
Query rewriting
Multi-query retrieval
Context compression
Reranking
Citation-aware generation
Conversation memory
Follow-up questions
Confidence estimation
---
Phase 7 — Production Engineering
Planned
Production-oriented improvements:
Alembic database migrations
Rate limiting
API versioning
Redis caching
Prometheus metrics
Grafana dashboards
Distributed tracing
Structured JSON logs
CI/CD
Automated Docker builds
Security hardening
Production deployment
Secrets management
Horizontal scaling
---
Phase 8 — Frontend
Planned
A frontend can eventually be added on top of the API.
Potential interface:
```text
+------------------------------------------------+
|              AI Knowledge Base                 |
+------------------------------------------------+
|                                                |
|  Documents                                     |
|  --------------------------------------------  |
|  Machine Learning                             |
|  Artificial Intelligence                      |
|  Deep Learning                                |
|                                                |
|  [ Add Document ]                              |
|                                                |
+------------------------------------------------+
|                                                |
|  Ask your knowledge base...                    |
|                                                |
|  [ What is machine learning? ] [Ask]          |
|                                                |
+------------------------------------------------+
|                                                |
|  Answer                                        |
|  --------------------------------------------  |
|  ...                                           |
|                                                |
|  Sources                                       |
|  - Machine Learning, chunk 8                  |
|  - Machine Learning, chunk 2                  |
|                                                |
+------------------------------------------------+
```
The frontend will remain separate from the backend so that the API remains independently usable.
---
Long-Term Architecture
The eventual system is intended to become:
```text
                         Client
                           |
                           v
                    API Gateway / FastAPI
                           |
              +------------+-------------+
              |                          |
              v                          v
       Authentication               Query Service
              |                          |
              v                          v
          PostgreSQL              Retrieval Pipeline
              |                          |
              |                  +-------+-------+
              |                  |               |
              |                  v               v
              |             Vector Search   Keyword Search
              |                  |               |
              |                  +-------+-------+
              |                          |
              |                       Reranker
              |                          |
              |                          v
              |                    Context Builder
              |                          |
              |                          v
              |                         LLM
              |                          |
              |                          v
              |                   Answer + Citations
              |
              |
       Document Management
              |
              v
       Background Workers
              |
       +------+------+
       |             |
       v             v
    Ingestion    Embeddings
       |             |
       +------+------+
              |
              v
          Vector DB
```
---
Engineering Goals
The long-term goals are:
Build a reliable RAG pipeline.
Understand retrieval instead of treating it as a black box.
Separate application data from vector data.
Keep components independently replaceable.
Introduce authentication and authorization.
Improve retrieval quality using measurable evaluation.
Process documents asynchronously.
Add production observability.
Containerize and deploy the system.
Build a complete AI backend rather than only an LLM wrapper.
---
Learning Goals
Backend Engineering
REST API design
Request validation
Service layers
Repository patterns
Error handling
Logging
Database integration
API architecture
AI Engineering
Embeddings
Vector databases
Semantic search
Retrieval pipelines
Context construction
RAG
LLM integration
Retrieval evaluation
Infrastructure
Docker
Docker Compose
PostgreSQL
Environment configuration
Service dependencies
Persistent storage
Software Engineering
Separation of concerns
Testing
Evaluation
Configuration management
Version control
Incremental development
---
Development Philosophy
The project is being developed incrementally.
Instead of immediately attempting to build a large production platform, the system is expanded layer by layer:
```text
Basic API
   ↓
Document ingestion
   ↓
Chunking
   ↓
Embeddings
   ↓
Vector retrieval
   ↓
RAG
   ↓
Persistent storage
   ↓
Source attribution
   ↓
Authentication
   ↓
Advanced retrieval
   ↓
Background processing
   ↓
Observability
   ↓
Deployment
```
Each layer is intended to build understanding of the engineering problem before introducing the next level of complexity.
---
Future Vision
The final goal is to transform this project from a RAG API into a complete personal/team knowledge platform where users can:
Create accounts
Upload or ingest knowledge
Organize documents into collections
Search their knowledge semantically
Ask questions about their data
Receive grounded answers
Inspect citations
Manage document ownership
Process large knowledge bases asynchronously
Monitor system performance
The backend will remain modular so that individual components such as the embedding model, vector database, retrieval strategy, or LLM provider can be replaced without rewriting the entire system.
---
License
This project is currently intended as a learning and portfolio project.
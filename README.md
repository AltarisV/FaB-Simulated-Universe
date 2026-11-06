# Flesh and Blood Simulated Universe

A Docker Compose application for exploring and extending the world of *Flesh and Blood*. It collects hero, story, and world information; stores it in ChromaDB; and uses OpenAI chat completions to generate character personalities, create stories, and answer questions grounded in the stored universe data.

## Architecture

The application is split into three Python/Flask services:

| Service | Port | Responsibility |
| --- | ---: | --- |
| `universe-creation` | 8080 | Web interface for collecting official heroes, stories, and world information from the Flesh and Blood website. |
| `universe-generation` | 8081 | Web interface for story and personality generation, and for asking questions about the stored universe. |
| `universe-persistence` | 8082 | HTTP API for importing and querying hero, story, and world documents, and for saving generated content. |
| `chroma_db` | 8083 → 8000 | ChromaDB container defined in the Compose configuration. |

The persistence service uses ChromaDB's embedded `PersistentClient`, saving its data under `/var/lib/chromadb`. The Compose file also starts a standalone ChromaDB container. The services communicate over a private Docker bridge network; the web interfaces and persistence API are published on the host ports shown above.

The persistence service configures two ChromaDB collections:

- `heroes` uses ChromaDB's default embedding function.
- `openai` uses OpenAI's `text-embedding-ada-002` embedding function.

The generation service retrieves relevant documents from ChromaDB and supplies them as context to the OpenAI Chat Completions API. For example, story generation combines selected heroes, regional information, and prior narratives to help maintain continuity.

## Technology

- Python 3.10/3.11 and Flask
- Docker and Docker Compose
- ChromaDB for vector storage and retrieval
- OpenAI API for text generation and embeddings
- Requests and Beautiful Soup for collecting source material

## Requirements

- Docker Desktop or Docker Engine with Docker Compose
- An OpenAI API key

## Run locally

1. Clone the repository and change to its root directory.
2. Create a `.env` file:

   ```dotenv
   OPENAI_API_KEY=your_openai_api_key
   APP_SECRET_KEY=replace_with_a_random_secret
   ```

3. Build and start the services:

   ```shell
   docker compose up --build -d
   ```

4. Import the sample hero collection into the default-embedding collection:

   ```shell
   curl.exe -X POST http://localhost:8082/heroes/importCollection/heroes
   ```

   To import the same sample data into the OpenAI-embedding collection:

   ```shell
   curl.exe -X POST http://localhost:8082/openai/importCollection/heroes
   ```

   Collection files are read from `services/simulated-universe/persistence/collections/`. The endpoint expects the filename without the `_collection.json` suffix.

5. Open the interfaces:

   - Data collection: <http://localhost:8080>
   - Story and personality generation, and Q&A: <http://localhost:8081>
   - Persistence API: <http://localhost:8082>
   - Standalone ChromaDB container: <http://localhost:8083>

The ChromaDB data is stored in the `chromadb-data` Docker volume and persists when the containers are stopped. To stop the services while retaining data, run `docker compose down`. Removing the volume will permanently delete the stored data.

## Main workflows

1. Use the creation interface to collect hero, story, and regional information from the official website.
2. The creation service sends the collected records to the persistence API, which stores them in ChromaDB with metadata for filtering.
3. Use the generation interface to generate a hero personality or a story. Relevant existing records are retrieved as context; generated stories can then be saved back to the collection.
4. Ask a question through the Q&A interface. The application retrieves matching records and asks the language model to answer using that context.

## Project layout

```text
services/simulated-universe/
├── creation/       # Flask UI and source-site scraping
├── generation/     # Flask UI and OpenAI-powered generation
└── persistence/    # Flask API, ChromaDB integration, and sample collections
docker-compose.yml  # Local multi-container setup
```

## Notes

- The application depends on the availability and structure of the source website and on access to the OpenAI API.
- OpenAI API usage may incur charges. Keep API keys in `.env`; do not commit credentials.
- This is a project prototype. Review configuration, API access controls, dependency versions, and model availability before deploying it to a public or production environment.

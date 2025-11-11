
# Knowledge Graph Builder
![Python](https://img.shields.io/badge/Python-yellow)
![FastAPI](https://img.shields.io/badge/FastAPI-green)
![React](https://img.shields.io/badge/React-blue)

Transform unstructured data (PDFs, DOCs, TXT, YouTube videos, web pages, etc.) into a structured Knowledge Graph stored in Neo4j using the power of Large Language Models (LLMs) and the LangChain framework.

This application allows you to upload files from various sources (local machine, GCS, S3 bucket, or web sources), choose your preferred LLM model, and generate a Knowledge Graph. 

---

## Key Features

### **Knowledge Graph Creation**
- Seamlessly transform unstructured data into structured Knowledge Graphs using advanced LLMs.
- Extract nodes, relationships, and their properties to create structured graphs.


### **Schema Support**
- Use a custom schema or existing schemas configured in the settings to generate graphs.

### **Graph Visualization**
- View graphs for specific or multiple data sources simultaneously in **Neo4j Bloom**.

### **Chat with Data**
- Interact with your data in the Neo4j database through conversational queries.
- Retrieve metadata about the source of responses to your queries.
- For a dedicated chat interface, use the standalone chat application with **[/chat-only](/chat-only) route.** 

### **LLMs Supported**
1. OpenAI
2. Gemini
3. Diffbot
4. Azure OpenAI(dev deployed version)
5. Anthropic(dev deployed version)
6. Fireworks(dev deployed version)
7. Groq(dev deployed version)
8. Amazon Bedrock(dev deployed version)
9. Ollama(dev deployed version)
10. Deepseek(dev deployed version)
11. Other OpenAI Compatible baseurl models(dev deployed version)
    
---

## Getting Started

### **Prerequisites**
- Neo4j Database **5.23 or later** with APOC installed.
  - **Neo4j Aura** databases (including the free tier) are supported.
  - If using **Neo4j Desktop**, you will need to deploy the backend and frontend separately (docker-compose is not supported).

---

## Deployment Options

### **Local Deployment**

#### Using Docker-Compose
Run the application using the default `docker-compose` configuration.

1. **Supported LLM Models**: 
   - By default, only OpenAI and Diffbot are enabled. Gemini requires additional GCP configurations. 
   - Use the `VITE_LLM_MODELS_PROD` variable to configure the models you need. Example:
     ```bash
     VITE_LLM_MODELS_PROD="openai_gpt_4o,openai_gpt_4o_mini,diffbot,gemini_1.5_flash"
     ```

2. **Input Sources**: 
   - By default, the following sources are enabled: `local`, `YouTube`, `Wikipedia`, `AWS S3`, and `web`. 
   - To add Google Cloud Storage (GCS) integration, include `gcs` and your Google client ID:
     ```bash
     VITE_REACT_APP_SOURCES="local,youtube,wiki,s3,gcs,web"
     VITE_GOOGLE_CLIENT_ID="your-google-client-id"
     ```

#### Chat Modes
Configure chat modes using the `VITE_CHAT_MODES` variable:
- By default, all modes are enabled: `vector`, `graph_vector`, `graph`, `fulltext`, `graph_vector_fulltext`, `entity_vector`, and `global_vector`. 
- To specify specific modes, update the variable. For example:
  ```bash
  VITE_CHAT_MODES="vector,graph"
  ```

---

### **Running Backend and Frontend Separately**

This guide helps you configure and run the frontend and backend of the application locally.

#### **Prerequisites**

- **Python 3.8+** with virtual environment
- **Node.js 18+** and **Yarn** (or npm)
- **Neo4j** running (pre-configured: `bolt://localhost:7687`, credentials `neo4j/password`)

#### **Backend Setup**

**1. Verify/Create `.env` file in backend**

The `.env` file should already be present in `backend/.env`. Verify it contains at least:

```env
# Neo4j Configuration (pre-configured for your Docker container)
NEO4J_URI=bolt://localhost:7687
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=password
NEO4J_DATABASE=neo4j

# Other optional configurations
# EMBEDDING_MODEL: Embedding model to use
#   - "all-MiniLM-L6-v2" (default, English only, 384 dim)
#   - "multilingual-minilm" (multilingual, 50+ languages, 384 dim) - Recommended for multilingual support
#   - "multilingual-mpnet" (multilingual, 50+ languages, 768 dim) - Higher quality
#   - "openai" (multilingual, 1536 dim) - Requires OPENAI_API_KEY
#   - "vertexai" (multilingual, 768 dim) - Requires Google Cloud credentials
EMBEDDING_MODEL=all-MiniLM-L6-v2
IS_EMBEDDING=TRUE
KNN_MIN_SCORE=0.94
UPDATE_GRAPH_CHUNKS_PROCESSED=20
NUMBER_OF_CHUNKS_TO_COMBINE=6
ENTITY_EMBEDDING=FALSE
GCS_FILE_CACHE=False
```

**2. Install Python dependencies**

```bash
cd backend

# If you don't have a virtual environment yet, create one
python -m venv venv

# Activate the virtual environment
# On macOS/Linux:
source venv/bin/activate
# On Windows:
# venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

**3. Start the Backend**

```bash
# Make sure you're in the backend directory
cd backend

# Start the server (port 8000 by default)
uvicorn score:app --reload --host 0.0.0.0 --port 8000
```

The backend will be available at: **http://localhost:8000**

You can verify it's working by visiting: **http://localhost:8000/health**

#### **Frontend Setup**

**1. Create `.env` file in frontend**

Create a `.env` file in the `frontend/` directory:

```bash
cd frontend
touch .env
```

Add to the `.env` file:

```env
# Backend API URL (port 8000)
VITE_BACKEND_API_URL=http://localhost:8000

# Other optional configurations (if needed)
VITE_REACT_APP_SOURCES=local,wiki,s3,youtube,web
VITE_SKIP_AUTH=true
```

**2. Install Node.js dependencies**

```bash
cd frontend

# Install dependencies (use yarn if available, otherwise npm)
yarn install
# or
npm install
```

**3. Start the Frontend**

```bash
# Make sure you're in the frontend directory
cd frontend

# Start the development server
yarn dev
# or
npm run dev
```

The frontend will be available at: **http://localhost:5173** (default Vite port)

#### **Complete Startup**

**Option 1: Two Separate Terminals (Recommended)**

**Terminal 1 - Backend:**
```bash
cd backend
source venv/bin/activate  # or venv\Scripts\activate on Windows
uvicorn score:app --reload --host 0.0.0.0 --port 8000
```

**Terminal 2 - Frontend:**
```bash
cd frontend
yarn dev
```

**Option 2: Automatic Startup Script**

You can create a script to start both simultaneously.

**For macOS/Linux** (`start.sh`):
```bash
#!/bin/bash

# Start backend in background
cd backend
source venv/bin/activate
uvicorn score:app --reload --host 0.0.0.0 --port 8000 &
BACKEND_PID=$!

# Wait for backend to be ready
sleep 3

# Start frontend
cd ../frontend
yarn dev &
FRONTEND_PID=$!

echo "Backend PID: $BACKEND_PID"
echo "Frontend PID: $FRONTEND_PID"
echo "Press Ctrl+C to stop both"

# Wait for user to press Ctrl+C
wait
```

Make executable: `chmod +x start.sh` and run: `./start.sh`

#### **Configuration Verification**

**1. Verify Backend**

Open your browser and visit:
- **Health Check**: http://localhost:8000/health
- You should see: `{"healthy": true}`

**2. Verify Frontend**

Open your browser and visit:
- **Frontend**: http://localhost:5173
- You should see the application interface

**3. Verify Neo4j Connection**

1. In the frontend, click the connection button
2. Enter:
   - Protocol: `bolt://`
   - Host: `localhost`
   - Port: `7687`
   - Database: `neo4j`
   - Username: `neo4j`
   - Password: `password`
3. Click "Connect"
4. You should see a successful connection message

#### **Troubleshooting**

**Backend won't start**

**Error: "Module not found"**
```bash
# Make sure you've installed all dependencies
cd backend
pip install -r requirements.txt
```

**Error: "Port already in use"**
```bash
# Change the port in the uvicorn command
uvicorn score:app --reload --host 0.0.0.0 --port 8001
# And update VITE_BACKEND_API_URL in frontend/.env
```

**Error: "Cannot connect to Neo4j"**
- Verify that the Neo4j container is running: `docker ps | grep knowledge-graph`
- Verify credentials in the `backend/.env` file
- Test the connection: `cypher-shell -a bolt://localhost:7687 -u neo4j -p password`

**Frontend won't start**

**Error: "Cannot find module"**
```bash
cd frontend
rm -rf node_modules
yarn install  # or npm install
```

**Error: "Cannot connect to backend"**
- Verify that the backend is running on http://localhost:8000
- Verify the `frontend/.env` file contains: `VITE_BACKEND_API_URL=http://localhost:8000`
- Restart the frontend after modifying the `.env` file

**Error: "Port 5173 already in use"**
```bash
# Vite will automatically use the next available port
# Or specify a different port by modifying vite.config.ts
```

**CORS Errors**

If you see CORS errors in the browser:
- The backend is already configured to allow CORS from any origin (`allow_origins=["*"]`)
- If they persist, verify that the backend is running and accessible

#### **Port Structure**

- **Backend**: `8000` (FastAPI/Uvicorn)
- **Frontend**: `5173` (Vite dev server)
- **Neo4j Bolt**: `7687` (pre-configured)
- **Neo4j HTTP**: `7474` (for Neo4j Browser, optional)

#### **Useful Commands**

**Stop Services**

**Backend**: Press `Ctrl+C` in the backend terminal

**Frontend**: Press `Ctrl+C` in the frontend terminal

**Check Processes**

```bash
# Check if backend is running
lsof -i :8000

# Check if frontend is running
lsof -i :5173

# Check if Neo4j is running
docker ps | grep knowledge-graph
```

**Logs and Debug**

**Backend**: Logs are printed directly in the terminal where you started uvicorn

**Frontend**: Open the browser console (F12) to see logs and errors

#### **Important Notes**

1. **`.env` files**: Make sure the `.env` files are in the correct directory:
   - `backend/.env` for backend
   - `frontend/.env` for frontend

2. **Virtual Environment**: Always remember to activate the virtual environment before starting the backend

3. **Dependencies**: If you add new dependencies, remember to update:
   - `backend/requirements.txt` for Python
   - `frontend/package.json` for Node.js

4. **Hot Reload**: Both servers support hot reload:
   - Backend: `--reload` flag in uvicorn
   - Frontend: Vite does it automatically

5. **Neo4j**: Make sure the Neo4j container is always running before starting the application

#### **Multilingual Support**

The system supports multilingual capabilities through multilingual embedding models. This allows you to:
- Load documents in different languages (Italian, French, German, Spanish, etc.)
- Query in one language and get results from documents in other languages
- It's no longer necessary for the query to be in the same language as the document

**Multilingual Configuration**

To enable multilingual support, modify the `backend/.env` file:

```env
# Option 1: Lightweight multilingual model (384 dimensions, 50+ languages)
EMBEDDING_MODEL=multilingual-minilm

# Option 2: High-quality multilingual model (768 dimensions, 50+ languages)
EMBEDDING_MODEL=multilingual-mpnet

# Option 3: OpenAI (multilingual, requires API key)
# Uses the text-embedding-3-small model (1536 dim) which supports 100+ languages
EMBEDDING_MODEL=openai
OPENAI_API_KEY=sk-your-openai-api-key-here

# Option 4: Vertex AI (multilingual, requires Google Cloud credentials)
EMBEDDING_MODEL=vertexai
```

**Available Models**

| Model | Languages Supported | Dimensions | Quality | Notes |
|-------|-------------------|------------|---------|-------|
| `multilingual-minilm` | 50+ | 384 | Good | Recommended for most cases |
| `multilingual-mpnet` | 50+ | 768 | Excellent | Better quality, slower |
| `openai` | 100+ | 1536 | Excellent | Uses text-embedding-3-small, requires OPENAI_API_KEY, paid |
| `vertexai` | 100+ | 768 | Excellent | Requires Google Cloud credentials |
| `all-MiniLM-L6-v2` | English only | 384 | Good | Default, only for English documents |

**Important Notes**

1. **Model Change**: If you change the embedding model after already loading documents, you will need to:
   - Recreate embeddings for all existing chunks
   - Recreate vector indexes in Neo4j

2. **Dimension Compatibility**: Make sure the new model's dimension is compatible with existing indexes, otherwise you'll need to recreate them.

3. **Performance**: Multilingual models are generally slower than monolingual models, but offer greater flexibility.

#### **Next Steps**

Once everything is running:
1. Open http://localhost:5173 in your browser
2. Connect to Neo4j using the configured credentials
3. Start loading documents and creating your Knowledge Graph!

---

### **Cloud Deployment**

Deploy the application on **Google Cloud Platform** using the following commands:

#### **Frontend Deployment**
```bash
gcloud run deploy dev-frontend \
  --source . \
  --region us-central1 \
  --allow-unauthenticated
```

#### **Backend Deployment**
```bash
gcloud run deploy dev-backend \
  --set-env-vars "OPENAI_API_KEY=<your-openai-api-key>" \
  --set-env-vars "DIFFBOT_API_KEY=<your-diffbot-api-key>" \
  --set-env-vars "NEO4J_URI=<your-neo4j-uri>" \
  --set-env-vars "NEO4J_USERNAME=<your-username>" \
  --set-env-vars "NEO4J_PASSWORD=<your-password>" \
  --source . \
  --region us-central1 \
  --allow-unauthenticated
```

---
## For local llms (Ollama)
1. Pull the docker imgage of ollama
```bash
docker pull ollama/ollama
```
2. Run the ollama docker image
```bash
docker run -d -v ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama
```
3. Execute any llm model ex llama3
```bash
docker exec -it ollama ollama run llama3
```
4. Configure  env variable in docker compose.
```env
LLM_MODEL_CONFIG_ollama_<model_name>
#example
LLM_MODEL_CONFIG_ollama_llama3=${LLM_MODEL_CONFIG_ollama_llama3-llama3,
http://host.docker.internal:11434}
```
5. Configure the backend API url
```env
VITE_BACKEND_API_URL=${VITE_BACKEND_API_URL-backendurl}
```
6. Open the application in browser and select the ollama model for the extraction.
7. Enjoy Graph Building.
---


## Usage
1. Connect to Neo4j Aura Instance which can be both AURA DS or AURA DB by passing URI and password through Backend env, fill using login dialog or drag and drop the Neo4j credentials file.
2. To differntiate we have added different icons. For AURA DB we have a database icon and for AURA DS we have scientific molecule icon right under Neo4j Connection details label.
3. Choose your source from a list of Unstructured sources to create graph.
4. Change the LLM (if required) from drop down, which will be used to generate graph.
5. Optionally, define schema(nodes and relationship labels) in entity graph extraction settings.
6. Either select multiple files to 'Generate Graph' or all the files in 'New' status will be processed for graph creation.
7. Have a look at the graph for individual files using 'View' in grid or select one or more files and 'Preview Graph'
8. Ask questions related to the processed/completed sources to chat-bot, Also get detailed information about your answers generated by LLM.

---


## ENV
| Env Variable Name       | Mandatory/Optional | Default Value | Description                                                                                      |
|-------------------------|--------------------|---------------|--------------------------------------------------------------------------------------------------|
|                                                                                                                                                                 |
| **BACKEND ENV** 
| OPENAI_API_KEY          | Mandatory           |            |An OpenAPI Key is required to use open LLM model to authenticate andn track requests                |
| DIFFBOT_API_KEY         | Mandatory           |            |API key is required to use Diffbot's NLP service to extraction entities and relatioship from unstructured data|
| BUCKET                  | Mandatory           |            |bucket name to store uploaded file on GCS                                                           |
| NEO4J_USER_AGENT        | Optional            | llm-graph-builder        | Name of the user agent to track neo4j database activity                              |
| ENABLE_USER_AGENT       | Optional            | true       | Boolean value to enable/disable neo4j user agent                                                   |
| DUPLICATE_TEXT_DISTANCE | Mandatory            | 5 | This value used to find distance for all node pairs in the graph and calculated based on node properties    |
| DUPLICATE_SCORE_VALUE   | Mandatory            | 0.97 | Node score value to match duplicate node                                                                 |
| EFFECTIVE_SEARCH_RATIO  | Mandatory            | 1 |                 |
| GRAPH_CLEANUP_MODEL     | Optional            | 0.97 |  Model name to clean-up graph in post processing                                                           |
| MAX_TOKEN_CHUNK_SIZE    | Optional            | 10000 | Maximum token size to process file content                                                               |
| YOUTUBE_TRANSCRIPT_PROXY| Optional            |   | Proxy key to process youtube video for getting transcript                                                   |
| EMBEDDING_MODEL         | Optional            | all-MiniLM-L6-v2 | Model for generating the text embedding (all-MiniLM-L6-v2 , openai , vertexai)                |
| IS_EMBEDDING            | Optional            | true          | Flag to enable text embedding                                                                    |
| KNN_MIN_SCORE           | Optional            | 0.94          | Minimum score for KNN algorithm                                                                  |
| GEMINI_ENABLED          | Optional            | False         | Flag to enable Gemini                                                                             |
| GCP_LOG_METRICS_ENABLED | Optional            | False         | Flag to enable Google Cloud logs                                                                 |
| NUMBER_OF_CHUNKS_TO_COMBINE | Optional        | 5             | Number of chunks to combine when processing embeddings                                           |
| UPDATE_GRAPH_CHUNKS_PROCESSED | Optional      | 20            | Number of chunks processed before updating progress                                        |
| NEO4J_URI               | Optional            | neo4j://database:7687 | URI for Neo4j database                                                                  |
| NEO4J_USERNAME          | Optional            | neo4j         | Username for Neo4j database                                                                       |
| NEO4J_PASSWORD          | Optional            | password      | Password for Neo4j database                                                                       |
| LANGCHAIN_API_KEY       | Optional            |               | API key for Langchain                                                                             |
| LANGCHAIN_PROJECT       | Optional            |               | Project for Langchain                                                                             |
| LANGCHAIN_TRACING_V2    | Optional            | true          | Flag to enable Langchain tracing                                                                  |
| GCS_FILE_CACHE          | Optional            | False         | If set to True, will save the files to process into GCS. If set to False, will save the files locally   |
| LANGCHAIN_ENDPOINT      | Optional            | https://api.smith.langchain.com | Endpoint for Langchain API                                                            |
| ENTITY_EMBEDDING        | Optional            | False         | If set to True, It will add embeddings for each entity in database |
| LLM_MODEL_CONFIG_ollama_<model_name>          | Optional      |               | Set ollama config as - model_name,model_local_url for local deployments |
| RAGAS_EMBEDDING_MODEL         | Optional      | openai              | embedding model used by ragas evaluation framework                               |
|                                                                                                                                                                        |
| **FRONTEND ENV** 
| VITE_BLOOM_URL               | Mandatory           | https://workspace-preview.neo4j.io/workspace/explore?connectURL={CONNECT_URL}&search=Show+me+a+graph&featureGenAISuggestions=true&featureGenAISuggestionsInternal=true | URL for Bloom visualization |
| VITE_REACT_APP_SOURCES       | Mandatory          | local,youtube,wiki,s3 | List of input sources that will be available                                               |
| VITE_CHAT_MODES              | Mandatory          | vector,graph+vector,graph,hybrid | Chat modes available for Q&A
| VITE_ENV                     | Mandatory          | DEV or PROD           | Environment variable for the app|
| VITE_LLM_MODELS              | Mandatory | 'diffbot,openai_gpt_3.5,openai_gpt_4o,openai_gpt_4o_mini,gemini_1.5_pro,gemini_1.5_flash,azure_ai_gpt_35,azure_ai_gpt_4o,ollama_llama3,groq_llama3_70b,anthropic_claude_3_5_sonnet' | Supported Models For the application
| VITE_BACKEND_API_URL         | Optional           | http://localhost:8000 | URL for backend API    |
| VITE_TIME_PER_PAGE          | Optional           | 50             | Time per page for processing                                                                    |
| VITE_CHUNK_SIZE              | Optional           | 5242880       | Size of each chunk of file for upload                                                                |
| VITE_GOOGLE_CLIENT_ID        | Optional           |               | Client ID for Google authentication                                                              |
| VITE_LLM_MODELS_PROD         | Optional      | openai_gpt_4o,openai_gpt_4o_mini,diffbot,gemini_1.5_flash | To Distinguish models based on the Enviornment PROD or DEV |
| VITE_AUTH0_CLIENT_ID | Mandatory if you are enabling Authentication otherwise it is optional |       |Okta Oauth Client ID for authentication
| VITE_AUTH0_DOMAIN | Mandatory if you are enabling Authentication otherwise it is optional |           | Okta Oauth Cliend Domain
| VITE_SKIP_AUTH | Optional | true | Flag to skip the authentication 
| VITE_CHUNK_OVERLAP | Optional |   20  | variable to configure chunk overlap
| VITE_TOKENS_PER_CHUNK | Optional |  100  | variable to configure tokens count per chunk.This gives flexibility for users who may require different chunk sizes for various tokenization tasks, especially when working with large datasets or specific language models.
| VITE_CHUNK_TO_COMBINE | Optional |   1  | variable to configure number of chunks to combine for parllel processing. 


## Links

[LLM Knowledge Graph Builder Application](https://llm-graph-builder.neo4jlabs.com/)

[Neo4j Workspace](https://workspace-preview.neo4j.io/workspace/query)

## Reference

[Demo of application](https://www.youtube.com/watch?v=LlNy5VmV290)

## Contact
For any inquiries or support, feel free to raise [Github Issue](https://github.com/neo4j-labs/llm-graph-builder/issues)


## Happy Graph Building!

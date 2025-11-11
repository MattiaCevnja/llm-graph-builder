# Guida Avvio Frontend e Backend in Locale

Questa guida ti aiuta a configurare e avviare il frontend e il backend dell'applicazione in locale.

## Prerequisiti

- **Python 3.8+** con virtual environment
- **Node.js 18+** e **Yarn** (o npm)
- **Neo4j** in esecuzione (già configurato: `bolt://localhost:7687`, credenziali `neo4j/password`)

## Configurazione Backend

### 1. Verifica/Crea file `.env` nel backend

Il file `.env` dovrebbe essere già presente in `backend/.env`. Verifica che contenga almeno:

```env
# Neo4j Configuration (già configurato per il tuo container Docker)
NEO4J_URI=bolt://localhost:7687
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=password
NEO4J_DATABASE=neo4j

# Altre configurazioni opzionali
EMBEDDING_MODEL=all-MiniLM-L6-v2
IS_EMBEDDING=TRUE
KNN_MIN_SCORE=0.94
UPDATE_GRAPH_CHUNKS_PROCESSED=20
NUMBER_OF_CHUNKS_TO_COMBINE=6
ENTITY_EMBEDDING=FALSE
GCS_FILE_CACHE=False
```

### 2. Installa le dipendenze Python

```bash
cd backend

# Se non hai ancora un virtual environment, crealo
python -m venv venv

# Attiva il virtual environment
# Su macOS/Linux:
source venv/bin/activate
# Su Windows:
# venv\Scripts\activate

# Installa le dipendenze
pip install -r requirements.txt
```

### 3. Avvia il Backend

```bash
# Assicurati di essere nella directory backend
cd backend

# Avvia il server (porta 8000 di default)
uvicorn score:app --reload --host 0.0.0.0 --port 8000
```

Il backend sarà disponibile su: **http://localhost:8000**

Puoi verificare che funzioni visitando: **http://localhost:8000/health**

## Configurazione Frontend

### 1. Crea file `.env` nel frontend

Crea un file `.env` nella directory `frontend/`:

```bash
cd frontend
touch .env
```

Aggiungi al file `.env`:

```env
# Backend API URL (porta 8000)
VITE_BACKEND_API_URL=http://localhost:8000

# Altre configurazioni opzionali (se necessario)
VITE_REACT_APP_SOURCES=local,wiki,s3,youtube,web
VITE_SKIP_AUTH=true
```

### 2. Installa le dipendenze Node.js

```bash
cd frontend

# Installa le dipendenze (usa yarn se disponibile, altrimenti npm)
yarn install
# oppure
npm install
```

### 3. Avvia il Frontend

```bash
# Assicurati di essere nella directory frontend
cd frontend

# Avvia il server di sviluppo
yarn dev
# oppure
npm run dev
```

Il frontend sarà disponibile su: **http://localhost:5173** (porta default di Vite)

## Avvio Completo

### Opzione 1: Due Terminali Separati (Consigliato)

**Terminale 1 - Backend:**
```bash
cd backend
source venv/bin/activate  # o venv\Scripts\activate su Windows
uvicorn score:app --reload --host 0.0.0.0 --port 8000
```

**Terminale 2 - Frontend:**
```bash
cd frontend
yarn dev
```

### Opzione 2: Script di Avvio Automatico

Puoi creare uno script per avviare entrambi contemporaneamente.

**Per macOS/Linux** (`start.sh`):
```bash
#!/bin/bash

# Avvia backend in background
cd backend
source venv/bin/activate
uvicorn score:app --reload --host 0.0.0.0 --port 8000 &
BACKEND_PID=$!

# Attendi che il backend sia pronto
sleep 3

# Avvia frontend
cd ../frontend
yarn dev &
FRONTEND_PID=$!

echo "Backend PID: $BACKEND_PID"
echo "Frontend PID: $FRONTEND_PID"
echo "Premi Ctrl+C per fermare entrambi"

# Attendi che l'utente prema Ctrl+C
wait
```

Rendi eseguibile: `chmod +x start.sh` e esegui: `./start.sh`

## Verifica della Configurazione

### 1. Verifica Backend

Apri il browser e visita:
- **Health Check**: http://localhost:8000/health
- Dovresti vedere: `{"healthy": true}`

### 2. Verifica Frontend

Apri il browser e visita:
- **Frontend**: http://localhost:5173
- Dovresti vedere l'interfaccia dell'applicazione

### 3. Verifica Connessione Neo4j

1. Nel frontend, clicca sul pulsante di connessione
2. Inserisci:
   - Protocollo: `bolt://`
   - Host: `localhost`
   - Porta: `7687`
   - Database: `neo4j`
   - Username: `neo4j`
   - Password: `password`
3. Clicca "Connect"
4. Dovresti vedere un messaggio di connessione riuscita

## Troubleshooting

### Backend non si avvia

**Errore: "Module not found"**
```bash
# Assicurati di aver installato tutte le dipendenze
cd backend
pip install -r requirements.txt
```

**Errore: "Port already in use"**
```bash
# Cambia la porta nel comando uvicorn
uvicorn score:app --reload --host 0.0.0.0 --port 8001
# E aggiorna VITE_BACKEND_API_URL nel frontend/.env
```

**Errore: "Cannot connect to Neo4j"**
- Verifica che il container Neo4j sia in esecuzione: `docker ps | grep knowledge-graph`
- Verifica le credenziali nel file `backend/.env`
- Testa la connessione: `cypher-shell -a bolt://localhost:7687 -u neo4j -p password`

### Frontend non si avvia

**Errore: "Cannot find module"**
```bash
cd frontend
rm -rf node_modules
yarn install  # o npm install
```

**Errore: "Cannot connect to backend"**
- Verifica che il backend sia in esecuzione su http://localhost:8000
- Verifica il file `frontend/.env` contiene: `VITE_BACKEND_API_URL=http://localhost:8000`
- Riavvia il frontend dopo aver modificato il file `.env`

**Errore: "Port 5173 already in use"**
```bash
# Vite userà automaticamente la porta successiva disponibile
# Oppure specifica una porta diversa modificando vite.config.ts
```

### CORS Errors

Se vedi errori CORS nel browser:
- Il backend è già configurato per permettere CORS da qualsiasi origine (`allow_origins=["*"]`)
- Se persistono, verifica che il backend sia in esecuzione e accessibile

## Struttura delle Porte

- **Backend**: `8000` (FastAPI/Uvicorn)
- **Frontend**: `5173` (Vite dev server)
- **Neo4j Bolt**: `7687` (già configurato)
- **Neo4j HTTP**: `7474` (per Neo4j Browser, opzionale)

## Comandi Utili

### Fermare i Servizi

**Backend**: Premi `Ctrl+C` nel terminale del backend

**Frontend**: Premi `Ctrl+C` nel terminale del frontend

### Verificare i Processi

```bash
# Verifica se backend è in esecuzione
lsof -i :8000

# Verifica se frontend è in esecuzione
lsof -i :5173

# Verifica se Neo4j è in esecuzione
docker ps | grep knowledge-graph
```

### Log e Debug

**Backend**: I log vengono stampati direttamente nel terminale dove hai avviato uvicorn

**Frontend**: Apri la console del browser (F12) per vedere i log e gli errori

## Note Importanti

1. **File .env**: Assicurati che i file `.env` siano nella directory corretta:
   - `backend/.env` per il backend
   - `frontend/.env` per il frontend

2. **Virtual Environment**: Ricorda di attivare sempre il virtual environment prima di avviare il backend

3. **Dipendenze**: Se aggiungi nuove dipendenze, ricorda di aggiornare:
   - `backend/requirements.txt` per Python
   - `frontend/package.json` per Node.js

4. **Hot Reload**: Entrambi i server supportano il hot reload:
   - Backend: `--reload` flag in uvicorn
   - Frontend: Vite lo fa automaticamente

5. **Neo4j**: Assicurati che il container Neo4j sia sempre in esecuzione prima di avviare l'applicazione

## Prossimi Passi

Una volta che tutto è in esecuzione:
1. Apri http://localhost:5173 nel browser
2. Connettiti a Neo4j usando le credenziali configurate
3. Inizia a caricare documenti e creare il tuo Knowledge Graph!


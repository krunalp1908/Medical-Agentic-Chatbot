# Medical Agentic Chatbot

MedGuide AI is a web-based medical education assistant. It combines retrieval from a local medical reference PDF (indexed in Pinecone), optional web search for current guidance, and OpenAI chat/vision/audio services. The API and web interface are built with FastAPI; LangGraph routes questions through retrieval and response generation and stores conversation checkpoints in SQLite.

> **Important:** This project is for educational and informational use only. It is not a medical device and does not provide a diagnosis, treatment plan, or substitute for a licensed clinician. AI-generated answers may be incomplete or wrong. For severe or emergency symptoms, contact local emergency services immediately.

## Contents

- [Features](#features)
- [How it works](#how-it-works)
- [Requirements](#requirements)
- [Setup](#setup)
- [Configuration](#configuration)
- [Index the medical reference](#index-the-medical-reference)
- [Run the application](#run-the-application)
- [API reference](#api-reference)
- [Data and privacy](#data-and-privacy)
- [Troubleshooting](#troubleshooting)
- [Project structure](#project-structure)

## Features

- Chat interface for medical education and symptom-related questions.
- Retrieval-augmented answers from the bundled `data/Medical_book.pdf`, with PDF page references.
- Optional Tavily web search when retrieved material is weak or a question asks for current guidance.
- Optional image input (JPEG, PNG, or WebP, up to 5 MB) for objective visual observations; image analysis is not a diagnosis.
- Optional microphone transcription and spoken answers.
- Conversation history and persistent LangGraph checkpoints in `medical_memory.db`.
- FastAPI health endpoint and interactive OpenAPI documentation.

The bundled Gale medical reference is from 2002 and only covers entries beginning A–B. Treat it as historical material, not current clinical guidance. Web search is optional and availability depends on a Tavily API key.

## How it works

1. The browser sends a question, recent chat history, and optionally an image to the FastAPI service.
2. If supplied, the image is sent to the configured OpenAI vision model to describe visible features.
3. The question and visual observations are embedded with OpenAI and searched against the Pinecone index.
4. LangGraph routes to optional Tavily web search if the local result is missing/weak or the question appears to need current guidance.
5. The configured OpenAI chat model produces an answer with available PDF page references or web URLs.
6. LangGraph checkpoints are stored in the local SQLite database `medical_memory.db`.

![Medical AI Agent architecture](docs/Medical%20AI%20Agent%20Architecture%20Infographic.png)

## Requirements

- Python **3.13 or newer** (the repository's `.python-version` and `pyproject.toml` specify Python 3.13).
- An OpenAI API key.
- A Pinecone API key and a Pinecone account that can create serverless indexes.
- A Tavily API key only if you want external web search.
- Network access to the configured providers.

The app dependencies are listed in `requirements.txt`. The `pyproject.toml` currently declares project metadata and the Python requirement, but does not declare these runtime dependencies; install from `requirements.txt`.

## Setup

Open PowerShell in the repository root (`Medical-Agentic-Chatbot`) and create/activate a virtual environment:

```powershell
py -3.13 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

If PowerShell blocks activation, either allow script activation for the current user according to your organization's policy, or skip activation and use the environment's Python directly:

```powershell
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
```

Create a `.env` file in the repository root. Do not commit this file or paste real keys into source control:

```dotenv
OPENAI_API_KEY=your-openai-api-key
PINECONE_API_KEY=your-pinecone-api-key
TAVILY_API_KEY=

PINECONE_INDEX=medical-agent-demo
OPENAI_CHAT_MODEL=gpt-4.1-mini
OPENAI_EMBED_MODEL=text-embedding-3-small
OPENAI_VISION_MODEL=gpt-4.1-mini

API_TOKEN=
ALLOWED_ORIGINS=http://localhost:8000,http://127.0.0.1:8000
```

`OPENAI_API_KEY` and `PINECONE_API_KEY` are required for chat and indexing. `TAVILY_API_KEY` is optional. The other values shown are defaults and can be omitted if unchanged. Keep `.env` private; rotate credentials if they are exposed.

## Configuration

| Variable | Required | Default | Purpose |
|---|---:|---|---|
| `OPENAI_API_KEY` | Yes | Empty | OpenAI embeddings, chat, optional image analysis, transcription, and speech. |
| `PINECONE_API_KEY` | Yes | Empty | Pinecone vector index access. |
| `TAVILY_API_KEY` | No | Empty | Enables web search for current or weakly supported questions. |
| `PINECONE_INDEX` | No | `medical-agent-demo` | Pinecone index name. The app creates it if missing, using 1536 dimensions, cosine similarity, and AWS `us-east-1`. |
| `OPENAI_CHAT_MODEL` | No | `gpt-4.1-mini` | Chat response model. |
| `OPENAI_EMBED_MODEL` | No | `text-embedding-3-small` | Text embedding model; the created index expects 1536-dimensional embeddings. |
| `OPENAI_VISION_MODEL` | No | `gpt-4.1-mini` | Model used for optional image observations. |
| `API_TOKEN` | No | Empty | Optional API access-token setting. |
| `ALLOWED_ORIGINS` | No | `http://localhost:8000,http://127.0.0.1:8000` | Comma-separated CORS origin allowlist. |

Environment variables are loaded from `.env` in the current working directory. Start the server from the repository root so the expected `.env` and SQLite database paths are used.

## Index the medical reference

Before starting the application for the first time, index the bundled PDF so it can answer questions using the local reference. With credentials configured, run this from the repository root:

```powershell
python ingest.py
```

To index another PDF:

```powershell
python ingest.py --pdf "C:\path\to\your-reference.pdf"
```

The ingestion process extracts text page by page, splits it into overlapping chunks, creates the configured Pinecone index if needed, generates embeddings, and upserts vectors. It makes paid/provider API calls and may take time for a large PDF. Re-running ingestion upserts matching chunks; changed or removed source content can leave older vectors in the index, so use a fresh index when you need a clean rebuild.

## Run the application

After indexing the reference, start the app from the repository root with the virtual environment active:

```powershell
python -m uvicorn app.main:app --reload
```

Open the web app at <http://127.0.0.1:8000>. The interactive API docs are at <http://127.0.0.1:8000/docs>; OpenAPI JSON is at <http://127.0.0.1:8000/openapi.json>.

Check that the server is responding:

```powershell
Invoke-RestMethod http://127.0.0.1:8000/api/health
```

The health response reports whether the required OpenAI and Pinecone keys are configured and whether Tavily is configured. It does not verify provider credentials, network connectivity, or Pinecone index readiness.

Stop the development server with **Ctrl+C**.

## API reference

All endpoints are served by the same FastAPI application. Request JSON endpoints accept JSON unless noted.

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/` | Serves the chat web interface. |
| `GET` | `/api/health` | Reports basic configuration status. |
| `POST` | `/api/chat` | Sends a question and returns an answer and source metadata. |
| `POST` | `/api/transcribe` | Transcribes an uploaded audio file. |
| `POST` | `/api/speak` | Returns synthesized speech as MPEG audio. |

### Chat

Example request:

```powershell
$body = @{
  message = "What are common causes of a headache?"
  history = @()
} | ConvertTo-Json -Depth 5

Invoke-RestMethod `
  -Uri http://127.0.0.1:8000/api/chat `
  -Method Post `
  -ContentType "application/json" `
  -Body $body
```

The JSON body accepts:

- `message` (string, required): 1–4,000 characters.
- `history` (array, optional): recent `{ "role": "user" | "assistant", "content": "..." }` messages. The API accepts at most 10 entries.
- `image_data` (string, optional): image data URL for a JPEG, PNG, or WebP image under 5 MB.

The response includes `answer`, `sources`, `tools`, and `conversation_id`. To continue a server-side checkpoint thread through the API, send the returned ID in the `X-Conversation-ID` request header on the next chat request. The browser UI also maintains recent chat history on the client.

### Audio

`POST /api/transcribe` accepts a multipart form upload with the field name `file`; audio uploads are limited to 10 MB. `POST /api/speak` accepts JSON with a `text` string and returns `audio/mpeg`; input text is truncated to 3,000 characters.

The web interface provides image upload/camera capture, microphone recording, and read-aloud controls. Browser camera and microphone access generally require localhost or a secure context and user permission.

## Data and privacy

- Questions, image content, and audio sent to the app may be processed by OpenAI; retrieval uses Pinecone; optional web searches use Tavily.
- Conversation checkpoints are persisted locally in `medical_memory.db`. Protect this file as user data and avoid sharing it if it contains sensitive conversation content.
- The app is not intended for protected health information. Do not submit identifying or sensitive medical information unless you have reviewed the applicable provider terms, privacy requirements, and deployment controls.
- The bundled PDF is historical reference material. Answers can be incorrect, and online search results are not independently guaranteed to be authoritative.
- Do not expose the development server to the public internet without implementing and verifying appropriate authentication, TLS, rate limiting, logging controls, and production deployment protections.

## Troubleshooting

| Symptom | Checks |
|---|---|
| Chat returns `503` | Set `OPENAI_API_KEY` and `PINECONE_API_KEY` in the root `.env`, then restart the server. |
| Chat returns `502` | Check server logs, provider credentials, outbound network access, Pinecone index availability, and model access. |
| Ingestion reports a missing PDF | Run from the repository root and verify `data/Medical_book.pdf` exists, or pass the correct path with `--pdf`. |
| Pinecone index creation fails | Confirm the API key, account permissions, index name, and availability of the configured AWS `us-east-1` serverless region. |
| Web search is unavailable | Set a valid `TAVILY_API_KEY`; without it the assistant continues without external results. |
| Camera or microphone does not work | Allow browser permission and use `localhost` or a secure HTTPS origin. |
| PowerShell cannot activate `.venv` | Use `.\.venv\Scripts\python.exe` directly, as shown in [Setup](#setup). |

## Project structure

```text
.
├── app/
│   ├── agent.py       # LangGraph retrieval, routing, response generation, SQLite checkpoint
│   ├── config.py      # Environment and .env settings
│   ├── knowledge.py   # OpenAI embeddings and Pinecone search/index helpers
│   └── main.py        # FastAPI routes and web application
├── data/
│   └── Medical_book.pdf
├── docs/
│   └── Medical AI Agent Architecture Infographic.png
├── static/            # Browser JavaScript and CSS
├── templates/         # HTML chat interface
├── ingest.py          # PDF text extraction and Pinecone indexing
├── requirements.txt
└── README.md
```
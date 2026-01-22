# Backend Environment Variables and Data Flow Documentation

## Overview
This document provides a comprehensive analysis of environment variables and data flow points in the NSUT Bot backend application.

---

## 1. Environment Variables (ENV Points)

### 1.1 Location of Environment Configuration
- **File to create:** `/backend/.env` (currently does not exist in repository)
- **Loading mechanism:** `load_dotenv()` is called at line 38 in `nsutbot.py`
- **Access pattern:** Uses `os.getenv()` and `os.environ.get()` throughout the code

### 1.2 Complete List of Environment Variables

| Variable | Type | Default Value | Required | Location | Purpose |
|----------|------|---------------|----------|----------|---------|
| `GOOGLE_CLIENT_ID` | String | None | Yes | Line 43 | Google OAuth 2.0 client ID for user authentication |
| `FRONTEND_URL` | String | `http://localhost:5173` | No | Line 44 | Frontend application URL for CORS configuration |
| `PINECONE_API_KEY` | String | None | **YES** | Line 45 | API key for Pinecone vector database (raises error if missing) |
| `BOT1` | String | None | Yes* | Line 65 | Groq API key for first bot instance |
| `BOT2` | String | None | Yes* | Line 66 | Groq API key for second bot instance |
| `BOT3` | String | None | Yes* | Line 67 | Groq API key for third bot instance |
| `BOT4` | String | None | Yes* | Line 68 | Groq API key for fourth bot instance |
| `BOT5` | String | None | Yes* | Line 69 | Groq API key for fifth bot instance |
| `PORT` | Integer | 3000 | No | Line 541 | Server port number |

**Note:** At least one BOT key (BOT1-BOT5) must be present. The system uses multiple keys for load balancing and rate-limit management.

### 1.3 Example .env File Template

```env
# Google OAuth Configuration
GOOGLE_CLIENT_ID=your_google_client_id_here.apps.googleusercontent.com

# Frontend Configuration
FRONTEND_URL=https://nsut-bot-personal.vercel.app

# Pinecone Vector Database (REQUIRED)
PINECONE_API_KEY=your_pinecone_api_key_here

# Groq API Keys (at least one required)
BOT1=your_groq_api_key_1
BOT2=your_groq_api_key_2
BOT3=your_groq_api_key_3
BOT4=your_groq_api_key_4
BOT5=your_groq_api_key_5

# Server Configuration
PORT=3000
```

---

## 2. Data Flow Points (Where Data is Going)

### 2.1 External Services Integration

#### A. **Pinecone Vector Database** ⭐ PRIMARY DATA STORE
- **Service:** Pinecone Serverless
- **Index Name:** `nsutbot-index`
- **Region:** AWS us-east-1
- **Authentication:** Via `PINECONE_API_KEY` environment variable
- **Data Operations:**
  - **WRITE** (Line 305): `index.upsert()` - Stores document embeddings with metadata
  - **READ** (Line 448): `index.query()` - Retrieval-Augmented Generation (RAG) semantic search
- **Data Structure Stored:**
  ```python
  {
    "id": "unique_id",
    "values": [embedding_vector_1024_dimensions],
    "metadata": {
      "text": "document_chunk",
      "filename": "uploaded_file.pdf",
      "user_email": "user@example.com"
    }
  }
  ```

#### B. **Groq AI API** ⭐ AI INFERENCE SERVICE
- **Service:** Groq Cloud LLM API
- **Authentication:** Rotating API keys (BOT1-BOT5) for rate-limit management
- **Models Used:**
  - Vision Model: `meta-llama/llama-4-scout-17b-16e-instruct` (Line 51)
  - Chat Model: `meta-llama/llama-4-scout-17b-16e-instruct` (Line 52)
  - Embedding Model: `llama-text-embed-v2` (Line 53)

**Data Sent to Groq:**
1. **Document Processing** (Line 280):
   - Base64-encoded images (PDF pages converted to PNG)
   - OCR/transcription prompts
   - Purpose: Extract text from documents

2. **Text Embeddings** (Line 191):
   - Text chunks for vectorization
   - Purpose: Generate 1024-dimensional embeddings for semantic search

3. **Chat Responses** (Line 521):
   - User queries
   - Retrieved context from Pinecone
   - Conversation history
   - Purpose: Generate AI chat responses (streamed to frontend)

#### C. **Google OAuth 2.0**
- **Service:** Google Identity Platform
- **Authentication:** Via `GOOGLE_CLIENT_ID` environment variable
- **Data Sent:** ID tokens from frontend
- **Operation:** `id_token.verify_oauth2_token()` (Line 337)
- **Data Received:** User email and name
- **Purpose:** User authentication and authorization

### 2.2 Data Flow to Frontend

#### Streaming Response Endpoint
- **Endpoint:** `/send` (POST)
- **Method:** `StreamingResponse` (FastAPI)
- **Data Format:** Server-Sent Events (SSE)
- **Content:** Real-time AI-generated chat messages
- **CORS Origins Allowed:**
  - `https://nsut-bot-personal.vercel.app` (Production)
  - `http://localhost:5173` (Development)

### 2.3 Complete Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    USER UPLOADS DOCUMENT                     │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
            ┌────────────────┐
            │ FastAPI Backend│
            │ /train endpoint│
            └────────┬───────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
        ▼            ▼            ▼
┌──────────┐  ┌──────────┐  ┌──────────┐
│  Groq AI │  │  Groq AI │  │ Pinecone │
│  Vision  │  │ Embedding│  │  Vector  │
│   (OCR)  │  │   API    │  │   Store  │
└────┬─────┘  └────┬─────┘  └────▲─────┘
     │             │              │
     └─────────────┴──────────────┘
           Text → Vectors → Store

┌─────────────────────────────────────────────────────────────┐
│                    USER SENDS CHAT QUERY                     │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
            ┌────────────────┐
            │ FastAPI Backend│
            │ /send endpoint │
            └────────┬───────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
        ▼            ▼            ▼
┌──────────┐  ┌──────────┐  ┌──────────┐
│  Groq AI │  │ Pinecone │  │  Groq AI │
│ Embedding│  │ Semantic │  │   Chat   │
│   API    │  │  Search  │  │   Model  │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │              │
     └─────────────┴──────────────┘
              RAG Pipeline
                     │
                     ▼
            ┌────────────────┐
            │    Frontend    │
            │ (Streaming SSE)│
            └────────────────┘
```

### 2.4 API Endpoints Summary

| Endpoint | Method | Data Sent | Data Received | External Service Called |
|----------|--------|-----------|---------------|------------------------|
| `/login` | POST | Google ID token | User session token | Google OAuth API |
| `/verify` | POST | Session token | User info | Internal (no external call) |
| `/train` | POST | Files (PDF/images), user email | Training status | Groq (Vision + Embedding), Pinecone |
| `/send` | POST | Chat message, user email | Streamed AI response | Groq (Embedding + Chat), Pinecone |
| `/get-image` | GET | Image filename | Image file | Internal (file system) |

---

## 3. Security & Configuration Notes

### 3.1 Required Actions for Production
1. Create `/backend/.env` file with all required environment variables
2. Obtain API keys from:
   - Pinecone (https://www.pinecone.io/)
   - Groq (https://groq.com/)
   - Google Cloud Console (for OAuth 2.0)
3. Set `FRONTEND_URL` to production domain
4. Ensure at least one Groq API key is configured (BOT1-BOT5)

### 3.2 Error Handling
- **Missing PINECONE_API_KEY:** Application raises `ValueError` and fails to start (Line 47-48)
- **No Groq API keys:** Application raises `ValueError` and fails to start (Line 73-74)
- **Missing GOOGLE_CLIENT_ID:** Authentication will fail at runtime (Line 337)

### 3.3 Rate Limiting Strategy
- Multiple Groq API keys (BOT1-BOT5) are used for load balancing
- Round-robin selection via `get_next_bot_client()` function (Line 78-83)
- Helps avoid rate limits on individual API keys

---

## 4. Third-Party Services Summary

| Service | Purpose | Data Type | Direction |
|---------|---------|-----------|-----------|
| **Pinecone** | Vector database for RAG | Embeddings + metadata | Bidirectional (read/write) |
| **Groq AI** | LLM inference (vision, chat, embeddings) | Text, images, prompts | Outbound (API calls) |
| **Google OAuth** | User authentication | ID tokens | Outbound (verification) |
| **Vercel** | Frontend hosting | None directly | Inbound (CORS allowed origin) |

---

## 5. Local File Storage

| Directory | Purpose | Files Stored |
|-----------|---------|--------------|
| `/backend/uploads/` | Temporary file uploads | User-uploaded PDFs and images |
| `/backend/uploads/images/` | Processed images | PNG conversions of PDF pages |

**Note:** Files are temporary and should be cleaned up after processing.

---

## Contact & Maintenance
For questions or updates to this documentation, please refer to the repository maintainers.

**Last Updated:** 2026-01-22

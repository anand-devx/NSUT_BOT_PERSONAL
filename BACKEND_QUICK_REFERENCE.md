# Quick Reference: Backend Environment & Data Flow

## 🔑 Environment Variables (9 total)

### Required (Must Set)
1. **PINECONE_API_KEY** - Vector database API key
2. **BOT1** (or BOT2-5) - At least one Groq API key
3. **GOOGLE_CLIENT_ID** - Google OAuth client ID

### Optional (Have Defaults)
4. **FRONTEND_URL** - Default: `http://localhost:5173`
5. **PORT** - Default: `3000`

### Load Balancing (Optional)
6-9. **BOT2, BOT3, BOT4, BOT5** - Additional Groq API keys for rate limiting

## 📍 Where Data Goes

### External Services (3 total)
1. **Pinecone** (Vector Database)
   - Stores: Document embeddings + metadata
   - Operations: Write (upsert), Read (query/search)

2. **Groq AI** (LLM Service)
   - Vision API: OCR/text extraction from documents
   - Embedding API: Convert text to vectors (1024 dimensions)
   - Chat API: Generate AI responses

3. **Google OAuth 2.0** (Authentication)
   - Verifies: User ID tokens from frontend
   - Returns: User email and name

### Frontend (Streaming)
- Real-time AI chat responses via Server-Sent Events (SSE)
- Allowed origins: Vercel production + localhost

## 🚀 Quick Setup

1. Copy template:
   ```bash
   cp backend/.env.example backend/.env
   ```

2. Fill in your API keys in `backend/.env`

3. Run backend:
   ```bash
   cd backend
   pip install -r requirements.txt
   python nsutbot.py
   ```

## 📚 Full Documentation
See `BACKEND_ENVIRONMENT_AND_DATA_FLOW.md` for comprehensive details.

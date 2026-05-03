---
globs:
  - "scripts/chatbot/**"
---

# Chatbot & Agentic RAG Rules

## Architecture
- Framework: LangGraph (multi-node agent graph)
- LLM: Google Gemini 2.0 Flash (`gemini-2.0-flash`)
- Vector DB: ChromaDB (local persistent)
- Query: Trino via PyTrino
- API: FastAPI

## Anti-Hallucination (QUAN TRỌNG)
- KHÔNG bịa đặt schema/table names — luôn tham chiếu ChromaDB
- Mọi response PHẢI có citation (source table, time range, layer)
- Nếu không có data → trả lời "Không tìm thấy dữ liệu" thay vì bịa

## Layer Selection Logic
```
time_period < 1 hour   → Fluss (hot)
1 hour ≤ time ≤ 7 days → Paimon (warm)
time_period > 7 days   → Iceberg (cold)
```

## Self-Correction
- Max retries: 3
- Log mỗi lần retry kèm error message
- Nếu fail sau 3 lần → trả lời user lỗi gì, không im lặng

## Files
- `app.py`: FastAPI entry point, agent graph definition
- `ingest.py`: Ingest schema metadata vào ChromaDB
- `rag_store.py`: ChromaDB wrapper (collection management)
- `download_model.py`: Model initialization

## Environment Variables
- `GEMINI_API_KEY`: Google Gemini API key (từ .env)
- `TRINO_HOST`: Trino coordinator host
- `TRINO_PORT`: Trino port (default 8080)
- `CHROMA_PERSIST_DIR`: ChromaDB storage path

## Detailed Implementation
Xem: `docs/agent-guides/agentic-rag.md`

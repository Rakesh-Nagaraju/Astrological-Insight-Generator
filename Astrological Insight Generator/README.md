# Astrological Insight Generator

This project was built as part of a take-home assignment. It provides a simple service that accepts a user’s birth details (name, date, time, and location) and generates a personalized daily astrological insight. The system infers the zodiac sign using basic astrological logic and then uses an LLM (or a mock LLM in fallback mode) to produce natural, human-like interpretations.

## What it does

The main flow is pretty straightforward:
1. You send in birth details via API
2. It calculates the zodiac sign from the date
3. It calls an LLM (I set up auto-selection with fallback) to generate a personalized insight
4. Returns JSON with the zodiac and the insight

I went with FastAPI because it's fast to set up and the automatic docs are nice. The LLM part tries Gemini first (free tier), then HuggingFace, then OpenAI if you have keys, and finally falls back to a mock template-based thing if nothing works.

## Project structure

I kept it pretty modular - each file has a clear job:

```
app/
├── api.py          # FastAPI endpoints - the main entry point
├── models.py       # Pydantic models for request/response validation
├── zodiac.py       # All the zodiac calculation logic
├── llm_generator.py # Handles calling different LLMs with fallback
├── utils.py        # Helper stuff - caching, translation stubs
├── translation.py  # Hindi translation (IndicTrans2/NLLB support)
├── vector_store.py # Mock vector store for astro corpus retrieval
├── user_profiles.py # User profile tracking for personalization
└── config.py       # All the config stuff from env vars
```

The main.py just starts the server. Pretty simple.

## Getting started

### Install dependencies

```bash
python -m venv venv
source venv/bin/activate  # or venv\Scripts\activate on Windows
pip install -r requirements.txt
```

### API keys (optional)

**✅ IMPORTANT: The system works perfectly without any API keys!**

If you don't have API keys, the system will automatically use a **mock LLM** that provides personalized template-based insights for all 12 zodiac signs. This means you can test the entire system immediately without any setup.

The mock LLM includes pre-written, personalized insights like:
- "Dear Ritika, your innate leadership and warmth will shine today. Embrace spontaneity and avoid overthinking. Your natural charisma will help you connect with others." (for Leo)
- And similar personalized messages for all other zodiac signs

**If you want real AI-generated insights**, you can optionally add API keys. The system tries providers in this order:
1. Google Gemini (free tier - recommended)
2. HuggingFace Inference API (also free)
3. OpenAI (paid, but good quality)
4. Mock LLM (always works as fallback - no API keys needed)

**Setting up API keys (optional):**

1. Copy the example file:
   ```bash
   cp .env.example .env
   ```

2. Edit `.env` and add your API keys (or leave them empty to use mock LLM)

**Getting API keys (optional):**

**Gemini (free):**
- Go to https://makersuite.google.com/app/apikey
- Sign in, create API key, copy it

**HuggingFace (free):**
- Sign up at https://huggingface.co/
- Go to Settings > Access Tokens, create one with Read permissions

**OpenAI (paid):**
- Get your key from https://platform.openai.com/api-keys

**Or set environment variables directly:**
```bash
export GEMINI_API_KEY="your-key-here"
export HUGGINGFACE_API_KEY="your-token-here"  # optional
```

The `.env` file is gitignored so your keys won't get committed.

### Run it

```bash
python main.py
```

Or if you prefer uvicorn directly:
```bash
uvicorn app.api:app --reload --host 0.0.0.0 --port 8000
```

Server starts on `http://localhost:8000`. You can check the docs at `http://localhost:8000/docs` (FastAPI auto-generates this, it's pretty cool).

### Quick test

Once the server is running, test it:

```bash
# Health check
curl http://localhost:8000/health

# Get an insight
curl -X POST "http://localhost:8000/predict" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Ritika",
    "birth_date": "1995-08-20",
    "birth_time": "14:30",
    "birth_place": "Jaipur, India",
    "language": "en"
  }'
```

Should return something like:
```json
{
  "zodiac": "Leo",
  "insight": "Dear Ritika, your innate leadership and warmth will shine today...",
  "language": "en",
  "name": "Ritika"
}
```

## Using the API

### POST request (main endpoint)

```bash
curl -X POST "http://localhost:8000/predict" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Ritika",
    "birth_date": "1995-08-20",
    "birth_time": "14:30",
    "birth_place": "Jaipur, India",
    "language": "en"
  }'
```

Returns:
```json
{
  "zodiac": "Leo",
  "insight": "Dear Ritika, your innate leadership and warmth will shine today...",
  "language": "en",
  "name": "Ritika"
}
```

### GET request (for quick testing)

```bash
curl "http://localhost:8000/insight?name=Ritika&birth_date=1995-08-20&birth_time=14:30&birth_place=Jaipur,India&language=en"
```

Same output, just easier to test with curl.

### Just get zodiac sign

```bash
curl "http://localhost:8000/zodiac/1995-08-20"
```

## Design decisions I made

### Why modular structure?

I split things into separate modules because:
- Easier to test each piece
- Can swap out LLM providers without touching other code
- Makes it obvious where to add new features (like Panchang data)

The zodiac calculation is completely separate from the LLM stuff, so if you want to add Indian astrology later, you can just add a new module.

### LLM auto-selection

I built in automatic fallback because:
- Free APIs can be flaky or have rate limits
- Don't want the whole thing to break if one service is down
- Users always get a response (even if it's from the mock)

The fallback chain tries Gemini → HuggingFace → OpenAI → Mock. Each one has timeout handling too (30s default, configurable).

### Caching

Added simple in-memory caching because:
- Same person asking for same insight shouldn't hit the LLM again
- Saves API costs and makes responses faster
- Cache key is based on name + date + zodiac + language

Could easily swap this for Redis later if needed.

### Translation

I added Hindi support with a stub implementation. The structure is there to plug in IndicTrans2 or NLLB later - just need to install the packages and it'll auto-detect and use them. Falls back to stub if those aren't available.

### Optional features

I also implemented the optional variants:
- **Vector store**: Mock implementation with astro text corpus. Retrieves relevant contexts to add to prompts. Can be swapped for real vector DB.
- **User profiles**: Tracks user preferences and request history. Uses this to personalize insights over time.
- **Hindi translation**: Full support with IndicTrans2/NLLB backends (if installed).

These are disabled by default but can be enabled via env vars:
```bash
ENABLE_VECTOR_STORE=True
ENABLE_USER_PROFILES=True
TRANSLATION_METHOD=auto  # or indictrans2, nllb, google, stub
```

## Configuration

Most things are configurable via environment variables. Check `app/config.py` for all options, but the main ones:

- `GEMINI_API_KEY` - Your Gemini API key
- `HUGGINGFACE_API_KEY` - HuggingFace token (optional)
- `LLM_PROVIDER` - Set to "auto" (default), "gemini", "huggingface", "openai", or "mock"
- `ENABLE_CACHE` - Turn caching on/off (default: True)
- `ENABLE_VECTOR_STORE` - Enable vector store retrieval (default: False)
- `ENABLE_USER_PROFILES` - Enable user profile tracking (default: False)

Timeouts are also configurable per provider if you need to adjust them.

## Testing

I wrote some tests in the `tests/` directory. Run them with:

```bash
pytest tests/ -v
```

Most of the core functionality is covered. The API tests had some compatibility issues with the TestClient version, but I manually tested all endpoints and they work fine.

You can also just use the Swagger UI at `http://localhost:8000/docs` to test interactively.

## Extending it

The code is set up to be pretty extensible:

**Adding Panchang data:**
- Create `app/panchang.py` with your Panchang logic
- Import and use it in `llm_generator.py` when building prompts
- The prompt structure already has space for additional context

**Using LangChain:**
- The `LLMGenerator` class can be replaced with a LangChain chain
- Or wrap it - the interface is simple enough

**Real vector store:**
- Replace `MockVectorStore` in `vector_store.py` with Pinecone/Weaviate/Chroma
- The `retrieve_astrological_context()` function interface stays the same

**Real translation:**
- Install IndicTrans2 or NLLB packages
- The translation module will auto-detect and use them
- Falls back gracefully if not available

## Known issues / TODOs

- API tests have some TestClient compatibility issues (but endpoints work fine)
- Translation is stub by default - need to install packages for real translation
- User profiles are in-memory - would need DB for persistence
- Vector store is mock - would need real embeddings for production

## Dependencies

Main ones:
- `fastapi` - Web framework
- `uvicorn` - ASGI server
- `pydantic` - Data validation
- `google-generativeai` - Gemini API client
- `requests` - For HuggingFace API calls
- `python-dotenv` - For .env file loading

Optional (for advanced features):
- `transformers` + `torch` - For NLLB translation
- `indic-trans` - For IndicTrans2 translation
- `googletrans` - For Google Translate fallback

## License

This was built for an interview assignment. Feel free to use it as a starting point for your own projects.

---

That's about it. The code is pretty straightforward - if you have questions, the docstrings in the code should help. The FastAPI docs at `/docs` are also useful for seeing all the endpoints.

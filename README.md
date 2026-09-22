# Text-to-SQL Agent

A LangChain-based agent that converts natural language questions into SQL queries, runs them against a SQLite database (the [Chinook](https://github.com/lerocha/chinook-database) sample database), and returns a natural-language answer.

## How it works

The pipeline has two stages, chained together with LangChain's LCEL (`RunnablePassthrough`):

1. **Question → SQL**: The user's question and the database schema are passed to an LLM, which generates a raw SQL query (no pre-amble, no markdown formatting).
2. **SQL → Answer**: The generated query is executed against the database, and the question, schema, query, and result are passed back to the LLM to produce a natural-language response.

```
question ─▶ [LLM: write SQL] ─▶ SQL query ─▶ [run against DB] ─▶ result
                                                                    │
                                                                    ▼
                                          [LLM: explain result] ─▶ answer
```

## Files

| File               | Description                                                                                   |
| ------------------ | --------------------------------------------------------------------------------------------- |
| `agent.ipynb`      | Main notebook — defines the schema loader, LLM setup, prompt chains, and runs example queries |
| `Chinook.db`       | Sample SQLite database (Artists, Albums, Tracks, Customers, Invoices, etc.)                   |
| `requirements.txt` | Python dependencies                                                                           |
| `_env.example`     | Template for the `.env` file with required API keys                                           |

## Setup

1. **Create a virtual environment** (recommended)

   ```bash
   python -m venv .venv
   source .venv/bin/activate   # on Windows: .venv\Scripts\activate
   ```

2. **Install dependencies**

   ```bash
   pip install -r requirements.txt
   ```

3. **Configure environment variables**

   Copy `_env.example` to `.env` and fill in your own keys:

   ```bash
   cp _env.example .env
   ```

   ```
   HUGGINGFACEHUB_API_TOKEN=your_huggingface_token_here
   GEMINI_API_KEY=your_gemini_api_key_here
   ```

   > ⚠️ **Note:** Never commit a real `.env` file or paste live API keys into example files. If a key has ever been exposed (e.g. committed to git or shared in a template), treat it as compromised and rotate it immediately on the provider's dashboard.

4. **Run the notebook**

   Open `agent.ipynb` in Jupyter and run the cells in order.

## Choosing an LLM

The `get_llm()` function supports two backends:

- **Google Gemini** (default): `ChatGoogleGenerativeAI(model="gemini-3.6-flash")` — requires `GEMINI_API_KEY`
- **Hugging Face**: `HuggingFaceEndpoint(repo_id="openai/gpt-oss-120b", ...)` — requires `HUGGINGFACEHUB_API_TOKEN`, used when calling `get_llm(load_from_hugging_face=True)`

```python
llm = get_llm(load_from_hugging_face=True)   # use Hugging Face
llm = get_llm()                              # use Gemini (default)
```

## Example usage

```python
from dotenv import load_dotenv
load_dotenv()

query = "Give me 10 Artists"
response = answer_user_query(query, llm=get_llm(load_from_hugging_face=True))
print(response.content)
```

### Example queries to try

The notebook includes a progression of query types worth testing against the schema:

1. **Straightforward**: "Give me the name of 10 Artists"
2. **Multiple columns**: "Give me the name and artist ID of 10 Artists"
3. **Foreign key lookups**: "Give me 10 Albums by the Artist with ID 1"
4. **Single join**: "Give some Albums by the Artist named Audioslave"
5. **Multi-level joins**: "Give some Tracks by the Artist named Audioslave"

## Performance evaluation

I evaluated the existing system against manually authored SQL using four
representative questions from the notebook: simple retrieval, multi-column
retrieval, a foreign-key filter, and a table join. The benchmark used the
local Chinook SQLite database and Python `time.perf_counter()` timings. The
application code was not modified for the evaluation.

| Metric                                  | Manual SQL |          Text-to-SQL |
| --------------------------------------- | ---------: | -------------------: |
| Successful queries                      | 4/4 (100%) |           4/4 (100%) |
| Average measured execution/task latency | 0.69575 ms |        1093.07202 ms |
| End-to-end measured latency difference  |          - | +1092.37627 ms/query |

The measured execution comparison isolates system latency: local SQL execution
is much faster because the Text-to-SQL path includes two LLM calls, one to
generate SQL and one to produce the natural-language answer.

To estimate the human authoring impact, I also applied an explicit 40 WPM
typing assumption to the actual benchmark text. Estimated average task time was
18.60070 seconds/query for typing and executing SQL manually versus 12.56807
seconds/query for entering a natural-language question and running the
Text-to-SQL pipeline. This corresponds to an estimated 6.03262 seconds saved
per query, or a 32.43% reduction. This estimate excludes SQL debugging,
validation, and result interpretation, and is not presented as a measured human
study.

The full methodology, input questions, raw timings, calculations, and excluded
quota-limited candidate query are available in
[`text2sql_impact.txt`](text2sql_impact.txt).

## Database schema

The Chinook database includes the following tables: `Album`, `Artist`, `Customer`, `Employee`, `Genre`, `Invoice`, `InvoiceLine`, `MediaType`, `Playlist`, `PlaylistTrack`, `Track`.

Schema is loaded via `SQLDatabase.from_uri("sqlite:///Chinook.db")` and injected into the LLM prompt at query time.

## Requirements

- Python 3.12+
- See `requirements.txt`:
  - `langchain`, `langchain-core`, `langchain-community`
  - `langchain-openai`, `langchain-google-genai`, `langchain-huggingface`
  - `python-dotenv`

## Known issues / notes

- `langchain-community` is deprecated and no longer actively maintained; consider migrating `SQLDatabase` usage to a standalone integration package when convenient.
- `langchain-openai` is listed as a dependency but the current notebook doesn't use `ChatOpenAI` — remove it from `requirements.txt` if it's not needed, or wire it into `get_llm()` as a third option.

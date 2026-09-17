# UdaPlay: AI Game Research Agent

UdaPlay is an AI research agent for answering questions about video games. It uses a local Retrieval-Augmented Generation (RAG) system as its first source of information and falls back to Tavily web search when the local evidence is missing or unreliable. Retrieved and newly discovered information can be parsed, retained in long-term memory, and returned as a structured report with source and confidence information.

Example questions include:

- Who developed FIFA 21?
- When was God of War Ragnarok released?
- What platform was Pokemon Red launched on?
- What is Rockstar Games working on right now?

## Project Goals

This project is completed in two connected parts:

1. Build a persistent ChromaDB vector store from the supplied game JSON files and implement semantic retrieval.
2. Build an agent that evaluates local retrieval, searches the web when necessary, maintains state, saves useful facts, and produces a clear answer.

## Build Steps

### 1. Prepare the development environment

Use Python 3.11 or newer. From the `project/starter` directory, create and activate a virtual environment, then install the packages required by the notebooks and library modules. Keep credentials in a local `.env` file and never commit that file.

```powershell
cd project/starter
py -3.11 -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install chromadb openai tavily-python python-dotenv jupyter
```

On macOS or Linux, activate the environment with `source .venv/bin/activate` instead.

### 2. Configure API credentials

Create `project/starter/.env` with the credentials required by the selected model, embeddings provider, and web search client:

```dotenv
OPENAI_API_KEY="your-openai-key"
CHROMA_OPENAI_API_KEY="your-chroma-or-embedding-key"
TAVILY_API_KEY="your-tavily-key"
```

Only `TAVILY_API_KEY` is needed for the web-search path, but the model and embedding keys are needed for the full application. Do not place real keys in notebooks or source files.

### 3. Inspect and normalize the game dataset

The `games/` directory contains fifteen JSON records. Read each file with the loader in `lib/loaders.py`, validate the expected fields, and convert each record into a consistent document. The indexed text should contain the game name, platform, genre, publisher, description, and release year so that searches can match both factual and descriptive questions. Preserve useful metadata, such as the source filename and game title, for citations and debugging.

### 4. Create the persistent ChromaDB vector store

Implement the reusable vector-store manager in `lib/vector_db.py`. It should create or open a persistent ChromaDB client, create a named collection with the configured embedding function, and add documents with stable IDs and metadata. Make indexing repeatable: rerunning the notebook should update or upsert records rather than create uncontrolled duplicates. Keep the database directory outside source control if it contains generated data.

### 5. Implement semantic retrieval and the RAG layer

Use `lib/rag.py` to expose a small retrieval API such as `retrieve_game(query, k)`. Embed the user query, retrieve the most relevant documents, and return the document text, metadata, and similarity or distance information. The retrieval result should retain enough provenance for the final report to identify whether an answer came from the local dataset. Handle an empty collection, malformed records, and no-result queries with useful errors or empty results instead of silently returning an invented answer.

### 6. Add the three agent tools

Implement and register the following tools using the project abstractions in `lib/tooling.py`:

- `retrieve_game`: searches the local ChromaDB collection and returns relevant game evidence.
- `evaluate_retrieval`: examines the retrieved evidence against the question, determines whether it is relevant and sufficient, and returns a confidence level plus a recommendation to answer locally or search the web.
- `game_web_search`: calls Tavily for current or missing information, returns the result text and URLs, and reports an actionable error when the API is unavailable.

Tool inputs and outputs should be structured and predictable. The evaluation tool must not treat a merely similar title as proof of an answer; low relevance, missing fields, conflicting evidence, or time-sensitive questions should trigger the web-search path.

### 7. Orchestrate the agent with state and memory

Use `lib/state_machine.py` and `lib/agents.py` to implement the workflow:

```text
user question
	-> retrieve_game
	-> evaluate_retrieval
	   -> sufficient: generate structured answer
	   -> insufficient: game_web_search
						  -> parse facts
						  -> persist memory
						  -> generate structured answer
```

Use `lib/memory.py` to persist useful facts and source metadata so later questions can benefit from previous research. Keep the current question, retrieved evidence, evaluation, web results, parsed facts, and final report in explicit state. Avoid storing unsupported claims, secrets, or raw responses that cannot be traced to a source.

### 8. Generate, test, and document the final report

Use the parser and message abstractions to produce a consistent response containing the answer, relevant game facts, confidence, and sources. Run the notebooks in order, test both local and fallback paths, and document any design choices or limitations. A complete implementation should answer known games from local data, use web search for missing or current facts, and retain source-backed discoveries for future requests.

## Project Structure

```text
project/
├── README.md
└── starter/
	├── games/                         # Seed game records in JSON format
	├── lib/
	│   ├── agents.py                  # Agent construction and orchestration
	│   ├── documents.py               # Document representations
	│   ├── evaluation.py              # Retrieval and answer evaluation
	│   ├── llm.py                     # Model client abstractions
	│   ├── loaders.py                 # JSON and document loading
	│   ├── memory.py                  # Long-term memory operations
	│   ├── messages.py                # Message and response types
	│   ├── parsers.py                 # Structured response parsing
	│   ├── rag.py                     # Retrieval-Augmented Generation logic
	│   ├── state_machine.py           # Workflow states and transitions
	│   ├── tooling.py                 # Agent tool definitions
	│   └── vector_db.py               # Persistent ChromaDB manager
	├── Udaplay_01_starter_project.ipynb # Part 1: RAG pipeline
	└── Udaplay_02_starter_project.ipynb # Part 2: agent implementation
```

## Getting Started

1. Open `project/starter/Udaplay_01_starter_project.ipynb` in VS Code or Jupyter.
2. Select the virtual-environment kernel and run the cells in order.
3. Complete the dataset loading, embedding, ChromaDB setup, and semantic-search exercises.
4. Open `project/starter/Udaplay_02_starter_project.ipynb` and use the same kernel.
5. Complete the tools, state-machine, fallback, memory, and reporting exercises.
6. Restart the kernel and run all cells from the beginning to verify that the workflow works on a clean session.

## Testing and Acceptance Criteria

There is no separate automated test suite supplied with the starter project. Validate the implementation through notebook cells and repeatable manual cases.

### Local RAG cases

- Ask about a game present in `games/`; confirm that `retrieve_game` returns the matching record and metadata.
- Ask for a release date, platform, genre, publisher, or description; confirm that the answer is grounded in retrieved fields.
- Repeat indexing; confirm that the collection does not gain duplicate documents.

### Fallback and evaluation cases

- Ask about a game absent from the local dataset; confirm that low retrieval confidence invokes `game_web_search`.
- Ask a current question, such as what a company is working on now; confirm that the agent uses web results and includes URLs.
- Simulate an empty result, weak match, conflicting result, or unavailable API; confirm that the agent reports the limitation rather than fabricating an answer.

### Memory and reporting cases

- Ask a follow-up question about a fact discovered through web search; confirm that persisted memory is consulted.
- Confirm that each final report includes an answer, confidence, evidence or sources, and a clear indication of whether the information came from local data, memory, or the web.
- Verify that the agent state contains the question, tool outputs, evaluation, and final report in the expected order.

## Required Deliverables

- A working persistent ChromaDB collection populated from the supplied JSON files.
- A reusable vector-store manager and semantic retrieval function.
- Implementations of `retrieve_game`, `evaluate_retrieval`, and `game_web_search`.
- A state-machine workflow with a local-first and web-fallback path.
- Source-aware structured output with confidence information.
- Long-term memory for parsed, source-backed facts.
- Completed Part 1 and Part 2 notebooks with representative test queries.
- Clear error handling for missing data, invalid records, unavailable APIs, and low-confidence retrieval.

## Built With

- [Python](https://www.python.org/) - Application language
- [ChromaDB](https://www.trychroma.com/) - Persistent vector database
- [OpenAI API](https://platform.openai.com/docs) - Language-model and embedding access
- [Tavily](https://tavily.com/) - Web search for fallback research
- [python-dotenv](https://github.com/theskumar/python-dotenv) - Environment variable loading
- [Jupyter](https://jupyter.org/) - Interactive project notebooks

## Security and Operational Notes

- Keep `.env` files and API keys local; rotate a key immediately if it is exposed.
- Treat web content as untrusted input and preserve URLs with any extracted facts.
- Use retrieval evaluation before generating an answer, especially for similarly named games.
- Do not claim that a fact is current unless it is supported by a current source.
- Generated ChromaDB data and long-term memory should be excluded from version control when they contain local or user-specific information.

## License

[License](../LICENSE.md)

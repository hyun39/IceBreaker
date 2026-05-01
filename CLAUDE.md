# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Ice Breaker is a Flask-based web application that generates personalized conversation starters by analyzing LinkedIn and Twitter profiles using LangChain and OpenAI.

## Development Commands

```bash
# Install dependencies
pipenv install

# Run the Flask application
pipenv run python app.py

# Format and lint
pipenv run black .
pipenv run isort .
pipenv run pylint [module_name]
```

## Architecture

Request flow: `app.py` (`POST /process`) → `ice_breaker.ice_break_with()` → agents discover profile URLs → third-party scrapers fetch data → three LangChain chains process it → structured Pydantic response.

**Agents** (`agents/`): ReAct agents using `gpt-4o-mini` that call Tavily search to find a person's LinkedIn and Twitter URLs from their name. Pulls the ReAct prompt from LangChain Hub (`hwchase17/react`).

**Chains** (`chains/custom_chains.py`): Three `PromptTemplate | ChatOpenAI | PydanticOutputParser` sequences using `gpt-3.5-turbo`. Summary and interests chains use `temperature=0`; ice breaker chain uses `temperature=1`.

**Output parsers** (`output_parsers.py`): `Summary`, `TopicOfInterest`, and `IceBreaker` Pydantic models, each with a `to_dict()` method used by the Flask JSON response.

**Third parties**:
- `linkedin.py`: calls Scrapin.io API. Has a `mock=True` flag that fetches a static GitHub Gist instead — useful for local dev without an API key.
- `twitter.py`: `scrape_user_tweets_mock()` fetches a static GitHub Gist. **`ice_breaker.py` always calls the mock**, never the real Twitter function. Note: `twitter.py` instantiates a `tweepy.Client` at module import time, so all five Twitter env vars must be present even when using the mock.

## Environment Variables

Required in `.env` (see `.env.example`):
- `OPENAI_API_KEY`
- `SCRAPIN_API_KEY` — Scrapin.io for LinkedIn data
- `TAVILY_API_KEY` — Tavily web search for agents
- `TWITTER_BEARER_TOKEN`, `TWITTER_API_KEY`, `TWITTER_API_KEY_SECRET`, `TWITTER_ACCESS_TOKEN`, `TWITTER_ACCESS_TOKEN_SECRET` — required at import time even when using the mock

Optional LangSmith tracing:
- `LANGCHAIN_TRACING_V2=true`, `LANGCHAIN_API_KEY`, `LANGCHAIN_PROJECT=ice_breaker`

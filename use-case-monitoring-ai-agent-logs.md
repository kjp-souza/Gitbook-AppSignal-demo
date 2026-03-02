---
description: >-
  Learn how to ship logs from a Python RAG agent to AppSignal Logging so you can
  track user queries, catch errors, and observe your AI application in
  production.
icon: head-side-gear
---

# Use Case: Monitoring AI Agent Logs

If you are building an AI application, knowing what your users are actually asking is just as important as knowing whether the app is running. This guide walks you through how to connect a Python RAG (Retrieval-Augmented Generation) agent to AppSignal Logging, so every user query shows up in your dashboard alongside your error tracking and performance data.

{% hint style="info" %}
This guide assumes you have an AppSignal account and have already followed the [Getting Started](/broken/pages/e98e1d78fd2f4351b8dcf14439572fa63178f6a9) guide to install and configure AppSignal in your Python project.
{% endhint %}

***

## What this agent does

The application in this guide is a FastAPI-based RAG agent. In plain terms, it works like this: you give it a folder of documents (PDFs, Markdown files, or YAML files), it reads and indexes them into a local vector database using FAISS, and then it answers user questions by retrieving the most relevant passages from that index before passing them to an LLM (OpenAI GPT-4 or a local Ollama model) to generate a response.

The interesting part for this guide is what happens in between: every time a user submits a question, the app logs that query to AppSignal. This means you can go into your AppSignal dashboard and see exactly what your users have been asking, when, and whether any of those requests resulted in an error.

***

## Requirements

Before you start, make sure the following packages are listed in your `requirements.txt` file.

```
appsignal
opentelemetry-instrumentation-fastapi
openai
requests
faiss-cpu
chromadb
langchain
python-dotenv
markdown
tiktoken
pyyaml
sentence-transformers
langchain-openai
langchain-community
```

Install them all at once by running the following command in your terminal.

```bash
pip install -r requirements.txt
```

***

## Setting your environment variables

The application needs a few keys to connect to AppSignal and OpenAI. Add the following to a `.env` file at the root of your project.

```dotenv
# Use the syslog key to ship logs to AppSignal
export APPSIGNAL_SYSLOG_API_KEY="ls-a1b2c3d4-e5f6-7890-abcd-ef12"
export OPENAI_API_KEY="sk-proj-XXXXXXXXXXXXXXXXXXXXXXXXXXXX"
export APPSIGNAL_APP_NAME="AI-RAG-agent"
export APPSIGNAL_APP_ENV="development"
```

{% hint style="warning" %}
**Use the right key for logging.** The `APPSIGNAL_SYSLOG_API_KEY` (prefixed `ls-`) is a log-specific key, separate from your server-side Push API key. You can find it under **Logging → Manage sources** in your AppSignal dashboard. Using the wrong key is the most common reason logs do not appear.
{% endhint %}

***

## Application code

The full `app.py` file below includes AppSignal initialization, a custom log handler that ships log records to AppSignal, the RAG pipeline, and the FastAPI routes. The comments explain the key integration points.

```python
# app.py
from fastapi import FastAPI, Request
from fastapi.templating import Jinja2Templates
from fastapi.responses import HTMLResponse
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from dotenv import load_dotenv
import os
import logging
import requests as http_requests
import time
import openai
import requests
import json
import yaml
import faiss
import chromadb
import datetime
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_community.vectorstores import FAISS
from langchain_community.document_loaders import PyPDFLoader, TextLoader
from langchain.schema import Document
from langchain_huggingface import HuggingFaceEmbeddings

# Import and start AppSignal before anything else
from __appsignal__ import appsignal
from appsignal import set_error
appsignal.start()

load_dotenv()
openai.api_key = os.getenv("OPENAI_API_KEY")

OLLAMA_URL = "http://localhost:11434/api/generate"
index = None


# --- AppSignal log handler ---
# This class extends Python's standard logging.Handler.
# Every time logger.info() or logger.error() is called, emit() fires
# and ships the record directly to AppSignal's logging endpoint.
class AppSignalLogHandler(logging.Handler):
    """Sends log records to AppSignal's logging endpoint."""
    def emit(self, record):
        try:
            api_key = os.getenv("APPSIGNAL_SYSLOG_API_KEY", "")
            if not api_key:
                return
            payload = {
                "log": [{
                    "timestamp": int(record.created),
                    "severity": record.levelname,
                    "message": self.format(record),
                    "attributes": {"group": "rag_agent"}
                }]
            }
            http_requests.post(
                "https://appsignal-endpoint.net/logs",
                json=payload,
                headers={"Authorization": f"Bearer {api_key}"},
                timeout=2
            )
        except Exception:
            # Never crash the app because of a logging failure
            pass


# --- Logger setup ---
# We attach two handlers: one prints to the terminal (console_handler)
# and the other ships to AppSignal (appsignal_handler).
logger = logging.getLogger("rag_agent")
logger.setLevel(logging.INFO)

formatter = logging.Formatter("%(asctime)s - %(name)s - %(levelname)s - %(message)s")

console_handler = logging.StreamHandler()
console_handler.setLevel(logging.INFO)
console_handler.setFormatter(formatter)
logger.addHandler(console_handler)

appsignal_handler = AppSignalLogHandler()
appsignal_handler.setFormatter(formatter)
logger.addHandler(appsignal_handler)


# --- FastAPI app ---
app = FastAPI()
templates = Jinja2Templates(directory="templates")

@app.get("/chat", response_class=HTMLResponse)
def chat_ui(request: Request):
    return templates.TemplateResponse("chat.html", {"request": request})


# --- Knowledge base helpers ---
def load_yaml(file_path):
    """Load a YAML file and convert it to a LangChain Document."""
    with open(file_path, 'r') as file:
        yaml_data = yaml.safe_load(file)

    def datetime_converter(obj):
        if isinstance(obj, datetime.datetime):
            return obj.isoformat()
        raise TypeError("Type not serializable")

    return Document(
        page_content=json.dumps(yaml_data, indent=2, default=datetime_converter),
        metadata={'source': file_path}
    )

def load_documents_from_directory(directory):
    """Load PDF, Markdown, plain text, and YAML files from a directory."""
    docs = []
    for root, _, files in os.walk(directory):
        for file in files:
            file_path = os.path.join(root, file)
            if file.endswith(".pdf"):
                loader = PyPDFLoader(file_path)
                docs.extend(loader.load())
            elif file.endswith((".md", ".txt")):
                loader = TextLoader(file_path)
                docs.extend(loader.load())
            elif file.endswith((".yaml", ".yml")):
                docs.append(load_yaml(file_path))
    return docs

def split_and_embed(docs):
    """Split documents into chunks and index them with FAISS."""
    global index
    text_splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=100)
    chunks = text_splitter.split_documents(docs)
    # Using an open-source embedding model to avoid OpenAI costs during indexing.
    # Swap to OpenAIEmbeddings() if you prefer.
    embeddings = HuggingFaceEmbeddings(model_name="BAAI/bge-base-en", model_kwargs={"device": "cpu"})
    index = FAISS.from_documents(chunks, embeddings)

def retrieve_context(query, k=3):
    """Return the top-k most relevant document chunks for a given query."""
    if index is None:
        print("No knowledge base loaded. Please upload documents first.")
        return []
    results = index.similarity_search(query, k=k)
    return "\n".join([doc.page_content for doc in results])


# --- LLM helpers ---
def ask_openai(prompt, context=""):
    """Send a prompt with retrieved context to OpenAI GPT-4."""
    try:
        response = openai.chat.completions.create(
            model="gpt-4",
            messages=[
                {"role": "system", "content": "You are a helpful research assistant."},
                {"role": "user", "content": f"Context:\n{context}\n\nQuestion: {prompt}"},
            ]
        )
        return response.choices[0].message.content
    except Exception as e:
        print(f"OpenAI request error: {e}")
        return None

def ask_ollama(prompt, model="mistral"):
    """Send a prompt to a locally running Ollama model."""
    payload = {"model": model, "prompt": prompt}
    try:
        response = requests.post(OLLAMA_URL, json=payload, stream=True)
        response.raise_for_status()
        full_response = ""
        for line in response.iter_lines():
            if line:
                try:
                    data = json.loads(line)
                    full_response += data.get("response", "")
                    if data.get("done", False):
                        break
                except json.JSONDecodeError as e:
                    print(f"JSON decoding error: {e}")
        return full_response
    except requests.RequestException as e:
        print(f"Ollama request error: {e}")
        return None


# --- Startup and routes ---
@app.on_event("startup")
def load_knowledge():
    """Load and index the knowledge base when the app starts."""
    global index
    docs = load_documents_from_directory(
        "/home/your-knowledge-base-directory"
    )
    split_and_embed(docs)
    print("Knowledge base loaded!")

@app.post("/ask")
def ask(question: str):
    """Handle a user question: log it, retrieve context, and return an answer."""
    try:
        if question == "crash":
            raise Exception("Test error from RAG agent")

        # This is the key line: every question is logged to AppSignal
        logger.info(f"User question: {question}")

        time.sleep(2)
        context = retrieve_context(question)
        answer = ask_openai(question, context)
        return {"answer": answer}
    except Exception as e:
        # Attaches the error to the active AppSignal trace before re-raising
        set_error(e)
        raise

# Instrument FastAPI with OpenTelemetry for performance tracing
FastAPIInstrumentor().instrument_app(app)
```

***

## Viewing your AI agent's logs in AppSignal

Once the app is running and receiving requests, your logs will appear in the AppSignal dashboard within a few seconds. To find them, follow these steps.

{% stepper %}
{% step %}
### Open your application

Open your AppSignal dashboard, select **Logging** and choose your application name from the list.
{% endstep %}

{% step %}
### Explore logs

Go to the **Explore** tab.
{% endstep %}

{% step %}
### Filter by time

Select a date range to filter the logs to the time window you want to inspect.
{% endstep %}

{% step %}
### Filter by time

Select a date range to filter the logs to the time window you want to inspect.
{% endstep %}

{% step %}
### Personalize your log view

Wrap log lines, show charts or edit the table for a better visualization experience.
{% endstep %}
{% endstepper %}

<figure><img src=".gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
**Tip:** Long log messages can be hard to read in the default view. Select the three-dot menu on the right side of the Explore tab and toggle **Wrap log lines** for a more readable layout.
{% endhint %}

You should now see one log entry per user question, with a timestamp, severity level, and the question text, all tagged under the `rag_agent` group.

***

## Next steps

Now that your agent's queries are visible in AppSignal, you can take this further in a few directions. You could add `logger.error()` calls to capture failed LLM responses alongside your error tracking. You could also add more attributes to the log payload (such as the model name or response latency) to make filtering more useful. If your agent grows to handle many users, combining logging with AppSignal's performance monitoring will give you a full picture of how the system behaves under load.

```
████████╗██╗  ██╗██████╗ ███████╗ █████╗ ████████╗    ██╗███╗   ██╗████████╗███████╗██╗
╚══██╔══╝██║  ██║██╔══██╗██╔════╝██╔══██╗╚══██╔══╝    ██║████╗  ██║╚══██╔══╝██╔════╝██║
   ██║   ███████║██████╔╝█████╗  ███████║   ██║       ██║██╔██╗ ██║   ██║   █████╗  ██║
   ██║   ██╔══██║██╔══██╗██╔══╝  ██╔══██║   ██║       ██║██║╚██╗██║   ██║   ██╔══╝  ██║
   ██║   ██║  ██║██║  ██║███████╗██║  ██║   ██║       ██║██║ ╚████║   ██║   ███████╗███████╗
   ╚═╝   ╚═╝  ╚═╝╚═╝  ╚═╝╚══════╝╚═╝  ╚═╝   ╚═╝       ╚═╝╚═╝  ╚═══╝   ╚═╝   ╚══════╝╚══════╝
                          ███╗   ███╗ █████╗ ████████╗██████╗ ██╗██╗  ██╗
                          ████╗ ████║██╔══██╗╚══██╔══╝██╔══██╗██║╚██╗██╔╝
                          ██╔████╔██║███████║   ██║   ██████╔╝██║ ╚███╔╝
                          ██║╚██╔╝██║██╔══██║   ██║   ██╔══██╗██║ ██╔██╗
                          ██║ ╚═╝ ██║██║  ██║   ██║   ██║  ██║██║██╔╝ ██╗
                          ╚═╝     ╚═╝╚═╝  ╚═╝   ╚═╝   ╚═╝  ╚═╝╚═╝╚═╝  ╚═╝
```

<div align="center">

### Local-First RAG for Incident Response, Threat Intelligence & Log Triage
##### Runs entirely inside a Google Colab notebook — no local install required

[![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?style=flat-square&logo=python&logoColor=white)](https://www.python.org/)
[![Google Colab](https://img.shields.io/badge/Runtime-Google_Colab-F9AB00?style=flat-square&logo=googlecolab&logoColor=white)](https://colab.research.google.com/)
[![LangChain](https://img.shields.io/badge/LangChain-RAG_Pipeline-1C3C3C?style=flat-square&logo=langchain&logoColor=white)](https://www.langchain.com/)
[![FAISS](https://img.shields.io/badge/FAISS-Vector_Store-0467DF?style=flat-square)](https://github.com/facebookresearch/faiss)
[![Gradio](https://img.shields.io/badge/Gradio-6.0-F97316?style=flat-square&logo=gradio&logoColor=white)](https://www.gradio.app/)
[![MITRE ATT&CK](https://img.shields.io/badge/MITRE-ATT%26CK-D4183D?style=flat-square)](https://attack.mitre.org/)
[![License](https://img.shields.io/badge/License-MIT-black?style=flat-square)](#license)

</div>

---

## Overview

**Threat Intel Matrix** is a localized, privacy-preserving Retrieval-Augmented Generation (RAG) pipeline built for **Tier 3 SOC Analysts**, **Incident Responders**, and **Penetration Testers**.

It exists to solve one specific problem: analysts frequently need to reason over incident response playbooks, MITRE ATT&CK technique data, and raw system logs — but pasting that material into a public-facing AI chatbot means leaking sensitive infrastructure details, internal procedures, or active incident data outside the organization's control.

Threat Intel Matrix keeps the retrieval layer isolated within a Google Colab notebook environment. Playbooks and MITRE ATT&CK data are embedded and indexed into a local FAISS vector store; raw logs are passed through the prompt at query time rather than being permanently written into the index. Analysts get grounded, cited, MITRE-aware answers without their logs or playbooks leaving their operational environment (aside from the inference call itself).

---

## Core Features

### Model & Embedding Layer
- **LLM:** `Qwen/Qwen2.5-72B-Instruct`, routed through the Hugging Face Serverless Inference API via a `ChatOpenAI`-compatible endpoint wrapper.
- **Embeddings:** `sentence-transformers/all-MiniLM-L6-v2` via `HuggingFaceEmbeddings`.
- **Vector Store:** Local `FAISS` index, held in Colab runtime memory — no external vector DB dependency.

### Ingestion Pipeline
- **Incident Response Playbooks** — Recursively loaded from `./incident-response-playbooks` using `DirectoryLoader` with `UnstructuredMarkdownLoader`. The root-level `README.md` is automatically pruned before vectorization to prevent index pollution from non-playbook content.
- **MITRE ATT&CK Framework** — Live fetch of the official STIX enterprise attack dataset (`enterprise-attack.json`), parsed to extract `attack-pattern` objects and map their names/descriptions directly into vector chunks alongside the playbook corpus.
- **Pasted Log Ingestion** — A dual-variable prompt schema (`context`, `logs`, `question`) allows raw JSON/EVTX log data to be submitted directly through the query interface at runtime, without permanently writing that log data into the FAISS index.

### Heuristic Guardrails & Reasoning Protocol
- **Primary Grounding** — The model is instructed to answer strictly from retrieved playbooks and MITRE data first.
- **Best-Effort Deduction** — Rather than returning "insufficient data" when no exact playbook match exists, the model performs bounded logical inference from the closest related concept in the retrieved context.
- **Mandatory Transparency Tagging** — Any portion of a response that relies on inference rather than direct retrieval is explicitly prefixed with `[WARNING: INFERRED STRATEGY]`, so analysts can immediately distinguish sourced guidance from deduced guidance.

### 1-Bit Retro Terminal Interface (Gradio 6.0)
- Built with `gr.Blocks` in a dual-column layout: raw system log input on the left, chat terminal output on the right.
- Custom monochrome CRT aesthetic via injected CSS — `Press Start 2P`, `VT323`, and `Share Tech Mono` Google Fonts, zero border-radius, hard-invert button hover states, and ASCII barcode section dividers.
- Built against Gradio 6.0+ event loops using dictionary-based chat history (`role` / `content` schema).

---

## Architecture Dataflow

```
                            ┌────────────────────────────┐
                            │   INGESTION (build-time)    │
                            └──────────────┬───────────────┘
                                            │
             ┌──────────────────────────────┼──────────────────────────────┐
             │                              │                              │
             ▼                              ▼                              │
 ┌─────────────────────┐      ┌──────────────────────────┐                 │
 │ IR Playbooks (.md)   │      │ MITRE ATT&CK STIX JSON   │                 │
 │ ./incident-response- │      │ enterprise-attack.json   │                 │
 │ playbooks/           │      │ (fetched live)           │                 │
 └──────────┬────────────┘      └──────────────┬────────────┘                 │
            │  DirectoryLoader +               │  Parse attack-pattern        │
            │  UnstructuredMarkdownLoader       │  objects → Document()        │
            │  (root README.md pruned)          │                              │
            ▼                                   ▼                              │
 ┌────────────────────────────────────────────────────────┐                   │
 │        RecursiveCharacterTextSplitter (chunking)         │                   │
 └───────────────────────────┬────────────────────────────┘                   │
                              ▼                                                │
 ┌────────────────────────────────────────────────────────┐                   │
 │  HuggingFaceEmbeddings (all-MiniLM-L6-v2) → FAISS Index  │◄──────────────────┘
 └───────────────────────────┬────────────────────────────┘
                              │
                              ▼
                     ┌──────────────────┐
                     │  FAISS Retriever  │
                     │   (k = 3 chunks)  │
                     └────────┬─────────┘
                              │
        ┌─────────────────────────────────────────────┐
        │              QUERY TIME (per request)         │
        └─────────────────────────────────────────────┘
                              │
   User Query ──────────────►│
   Pasted Logs (optional) ──►│  RunnableParallel
                              │  { context, question, logs }
                              ▼
                   ┌────────────────────────┐
                   │   PromptTemplate         │
                   │   + Reasoning Protocol   │
                   │   + Transparency Rules   │
                   └────────────┬─────────────┘
                                ▼
                   ┌────────────────────────┐
                   │  Qwen2.5-72B-Instruct    │
                   │  (HF Router / ChatOpenAI)│
                   └────────────┬─────────────┘
                                ▼
                   ┌────────────────────────┐
                   │   StrOutputParser        │
                   └────────────┬─────────────┘
                                ▼
                   ┌────────────────────────┐
                   │  Gradio Terminal UI      │
                   │  (grounded / [INFERRED]) │
                   └────────────────────────┘
```

---

## Requirements & Deployment

This project is built specifically to be deployed and executed within a **Google Colab notebook environment**, eliminating the need for local desktop dependencies.

- A Google account with access to [Google Colab](https://colab.research.google.com/)
- A Hugging Face account with an active [API token](https://huggingface.co/settings/tokens) (requires Serverless Inference API access)

---

## Installation (Google Colab Setup)

1. Open a new Google Colab notebook.
2. Securely store your Hugging Face API key: click the **Secrets** (key icon) in the left sidebar, add a new secret named `HUGGINGFACEHUB_API_TOKEN`, paste your key as the value, and enable **Notebook access**.
3. Create a new code cell and run the following to install dependencies and pull the playbook repository:

```python
# Install dependencies silently
!pip install -q langchain langchain-community langchain-core langchain-text-splitters langchain-huggingface langchain-openai faiss-cpu sentence-transformers unstructured markdown

# Clone the playbooks repository (ignores error if already cloned)
!git clone https://github.com/LetsDefend/incident-response-playbooks.git || true

# Clean up the root README to prevent vector database pollution
!rm -f incident-response-playbooks/README.md
```

---

## Usage

Paste the full implementation code (below) into a new cell in your Colab notebook and execute it. The Gradio 6.0 interface will launch directly beneath the cell inside your notebook.

- **Left panel** — paste raw JSON/EVTX system logs relevant to your current investigation (optional).
- **Right panel** — the terminal chat window. Enter a natural-language query and press `EXECUTE` or hit enter.
- Responses grounded in your playbooks or MITRE ATT&CK data are returned directly.
- Any response segment relying on model inference beyond the retrieved context is explicitly flagged with `[WARNING: INFERRED STRATEGY]`.

---

## Implementation Code

<details>
<summary><strong>Click to expand full source</strong></summary>

```python
# ==========================================
# 1. SETUP & DATA INGESTION
# ==========================================
import os
import getpass
import requests
from langchain_community.document_loaders import DirectoryLoader, UnstructuredMarkdownLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_community.vectorstores import FAISS
from langchain_huggingface import HuggingFaceEmbeddings
from langchain_openai import ChatOpenAI
from langchain_core.prompts import PromptTemplate
from langchain_core.runnables import RunnableParallel, RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser
from langchain_core.documents import Document
from google.colab import userdata

# Set Token securely from Colab Secrets
hf_token = userdata.get("HUGGINGFACEHUB_API_TOKEN")
os.environ["HUGGINGFACEHUB_API_TOKEN"] = hf_token

# Initialize Embeddings
embeddings = HuggingFaceEmbeddings(model_name="sentence-transformers/all-MiniLM-L6-v2")

# Ingest IR Playbooks
playbook_loader = DirectoryLoader(
    './incident-response-playbooks', 
    glob="**/*.md", 
    loader_cls=UnstructuredMarkdownLoader
)
playbook_docs = playbook_loader.load()
text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=150)
playbook_chunks = text_splitter.split_documents(playbook_docs)
vector_store = FAISS.from_documents(playbook_chunks, embeddings)

# Fetch MITRE ATT&CK Framework
mitre_url = "https://raw.githubusercontent.com/mitre-attack/attack-stix-data/master/enterprise-attack/enterprise-attack.json"
response = requests.get(mitre_url)
mitre_data = response.json()

mitre_documents = []
for obj in mitre_data.get('objects', []):
    if obj.get('type') == 'attack-pattern':
        name = obj.get('name', 'Unknown')
        description = obj.get('description', 'No description available.')
        mitre_documents.append(
            Document(page_content=f"MITRE Technique: {name}\nDescription: {description}")
        )

mitre_chunks = text_splitter.split_documents(mitre_documents)
vector_store.add_documents(mitre_chunks)
retriever = vector_store.as_retriever(search_kwargs={"k": 3})

# Initialize LLM
llm = ChatOpenAI(
    model="Qwen/Qwen2.5-72B-Instruct", 
    api_key=hf_token,
    base_url="https://router.huggingface.co/v1",
    max_tokens=512,
    temperature=0.1
)

# System Prompt with Deduction Protocol
template = """You are an expert Tier 3 SOC Analyst and Incident Responder.
You will be provided with retrieved security playbooks and raw system logs.

SECURITY GUARDRAILS & REASONING PROTOCOL:
1. Primary Grounding: Always attempt to answer the question using ONLY the provided playbooks and logs first.
2. Best-Effort Deduction: If the provided context lacks an exact match or is incomplete, DO NOT output "INSUFFICIENT DATA." Instead, use logical deduction to find the closest related concept in the context, and synthesize the most accurate, actionable recommendation possible.
3. Mandatory Transparency: If you are relying on logical deduction, partial matches, or outside cybersecurity knowledge to bridge gaps in the provided context, you MUST begin that specific recommendation with: "[WARNING: INFERRED STRATEGY]".
4. Log Analysis: If analyzing logs, explicitly cite the timestamp, host, user, and command. 

=== RETRIEVED PLAYBOOKS & INTEL ===
{context}

=== RAW SYSTEM LOGS (USER PROVIDED) ===
{logs}

=== QUESTION ===
{question}

Answer:"""

def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

def get_query(x): return x.get("question", "") if isinstance(x, dict) else str(x)
def get_logs(x): return x.get("logs", "No logs provided.") if isinstance(x, dict) else "No logs provided."

rag_chain = (
    RunnableParallel(
        context=lambda x: format_docs(retriever.invoke(get_query(x))),
        question=get_query,
        logs=get_logs,
    )
    | PromptTemplate.from_template(template)
    | llm
    | StrOutputParser()
)

# ==========================================
# 2. GRADIO 6.0 MONOCHROME TERMINAL UI
# ==========================================
import gradio as gr

custom_css = """
@import url('https://fonts.googleapis.com/css2?family=Press+Start+2P&family=VT323&family=Share+Tech+Mono&display=swap');

:root {
    --background-fill-primary: #000000 !important;
    --background-fill-secondary: #000000 !important;
    --text-color: #ffffff !important;
    --border-color-primary: #ffffff !important;
    --body-text-color: #ffffff !important;
}

body, .gradio-container {
    background-color: #000000 !important;
    color: #ffffff !important;
    font-family: 'Share Tech Mono', monospace !important;
}

* { border-radius: 0px !important; box-shadow: none !important; }

h1, h2, h3 {
    font-family: 'Press Start 2P', monospace !important;
    text-transform: uppercase;
    color: #ffffff !important;
    text-align: center;
    letter-spacing: 2px;
}

textarea, input, .gr-box, .gr-chatbot {
    font-family: 'VT323', monospace !important;
    font-size: 1.3rem !important;
    border: 1px solid #ffffff !important;
    background-color: #000000 !important;
    color: #ffffff !important;
}

.message, [data-testid="user"], [data-testid="bot"], .message-wrap {
    background-color: #000000 !important;
    background: #000000 !important;
    border: 1px solid #ffffff !important;
    color: #ffffff !important;
}

.message p { color: #ffffff !important; }

button {
    font-family: 'Press Start 2P', monospace !important;
    font-size: 0.7rem !important;
    border: 1px solid #ffffff !important;
    background-color: #000000 !important;
    color: #ffffff !important;
    transition: none !important;
}

button:hover {
    background-color: #ffffff !important;
    color: #000000 !important;
    cursor: pointer;
}

.barcode {
    font-family: 'VT323', monospace;
    letter-spacing: -2px;
    font-size: 1.5rem;
    text-align: center;
    user-select: none;
    color: #ffffff;
    margin: 15px 0;
}
"""

with gr.Blocks() as demo:
    gr.Markdown("# THREAT INTEL MATRIX")
    gr.Markdown("<div class='barcode'>||||| ||| || |||| ||||| ||| || |||| ||||| |||</div>")
    
    with gr.Row():
        with gr.Column(scale=1):
            gr.Markdown("### SYSTEM LOGS")
            logs_input = gr.Textbox(
                lines=18, 
                show_label=False, 
                placeholder="[+] PASTE RAW JSON/EVTX LOGS HERE...\n[+] LEAVE BLANK FOR GENERAL QUERIES..."
            )
        
        with gr.Column(scale=2):
            gr.Markdown("### TERMINAL")
            chatbot = gr.Chatbot(height=410, show_label=False)
            
            with gr.Row():
                query_input = gr.Textbox(lines=1, show_label=False, placeholder="[+] ENTER QUERY...", scale=4)
                submit_btn = gr.Button("EXECUTE", scale=1)
                
    gr.Markdown("<div class='barcode'>||||| ||| || |||| ||||| ||| || |||| ||||| |||</div>")

    def chat_logic(user_query, logs, history):
        if not user_query:
            return "", history
        
        dict_input = {
            "question": user_query,
            "logs": logs if logs.strip() else "No logs provided."
        }
        
        history.append({"role": "user", "content": user_query})
        history.append({"role": "assistant", "content": "Processing query against local vectors..."})
        yield "", history
        
        response = rag_chain.invoke(dict_input)
        history[-1]["content"] = response
        yield "", history

    submit_btn.click(fn=chat_logic, inputs=[query_input, logs_input, chatbot], outputs=[query_input, chatbot])
    query_input.submit(fn=chat_logic, inputs=[query_input, logs_input, chatbot], outputs=[query_input, chatbot])

demo.launch(debug=True, css=custom_css, theme=gr.themes.Monochrome())
```

</details>

---

## Roadmap

Planned future enhancements — **not yet implemented**:

| # | Feature | Description |
|---|---------|--------------|
| 1 | **Continuous Automated Log Ingestion** | A directory-watching pipeline to automatically detect and parse new `.json` / `.evtx` log dumps, extracting telemetry into natural-language documents without manual paste-in. |
| 2 | **Agentic Tool Calling** | Autonomous LangChain agents capable of live external lookups (e.g. VirusTotal hash checks, AlienVault OTX IP reputation queries) when suspicious artifacts are identified in submitted logs. |
| 3 | **Semantic Query Routing** | A router layer to filter non-security or off-topic queries before they reach vector retrieval, reducing token overhead and response latency. |
| 4 | **Automated Benchmark Evaluation** | Integration of the `RAGAS` framework to programmatically score Faithfulness, Answer Relevance, and Context Recall — driving toward zero untagged hallucinations. |

---

## License

Released under the [MIT License](#license).

---

<div align="center">

`[ SYSTEM READY ]` &nbsp;&nbsp;•&nbsp;&nbsp; Built for analysts who can't paste incident data into a public model.

</div>

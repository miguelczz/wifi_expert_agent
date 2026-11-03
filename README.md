<div align="center">

# WiFi Expert Agent

**Multi-agent, multimodal Wi-Fi network analyzer that turns Wireshark packet captures into precise AI-driven diagnostics.**

> 🔒 **Confidentiality & Intellectual Property Notice:**
> The source code for this project is private and owned by **Tigo Colombia**. This repository serves only as a **Technical Architecture Showcase** to demonstrate the system's functionality, observability patterns, and engineering design.

</div>

**English** · [Español](README.es.md)

---

## What does the WiFi Expert Agent do?

The WiFi expert agent receives network capture files (`.pcap` / `.pcapng`) generated with **Wireshark** and processes them through a specialized AI agent system that evaluates 802.11k/v/r protocol behavior (Band Steering / WiFi Roaming):

- ✅ **Automatic Band Steering analysis** — evaluates BTM Support, Association, Effective Steering, and KVR Standards
- ✅ **Intelligent diagnostics** — Groq (Llama 3.3), Gemini 2.0 Flash, and GPT-4o with dynamic routing by cost and quality
- ✅ **PDF report generation** — delivery-ready, with signal charts, BTM events, and technical conclusions
- ✅ **Contextual chat** — the Pipe assistant answers questions about the active analysis or WiFi technical documentation
- ✅ **Batch analysis** — processes dozens of captures in parallel and allows downloading results as a ZIP or executive PDF
- ✅ **Real-time observability** — tracks cost, latency, and LLM traces per model and operation

---

## 🖼️ Screenshots

### 1. Trace Explorer — Upload and Processing

> The user uploads a `.pcapng` file, selects the vendor, project, and client MAC address. The system processes the capture with **tshark** and generates the AI-powered analysis.

<div align="center">
<img src="./assets/upload.png" alt="Trace Explorer" width="800">
</div>

---

### 2. Analysis Result — Band Steering

> Immediate analysis result: technical compliance verdict per criterion (BTM Support, Association, Effective Steering, KVR Standards).

<div align="center">
<img src="./assets/band_steering.png" alt="Band Steering Result" width="800">
</div>

---

### 3. Band Change Visualization

> Interactive RSSI signal chart over time, with BTM event markers, band transitions (2.4 GHz / 5 GHz), and configurable thresholds.

<div align="center">
<img src="./assets/band_chart.png" alt="Band Change Chart" width="800">
</div>

---

### 4. Wireshark Packet List — Packet View

> Integrated panel showing the 802.11 packets extracted from the capture: source, destination, protocol, band, and frame description.

<div align="center">
<img src="./assets/packet_list.png" alt="Wireshark Packet List" width="800">
</div>

---

### 5. Generated PDF Report

> Automatically generated technical report: executive summary, technical analysis, steering metrics, BSSID detection, KVR standards, and final conclusion.

<div align="center">
<img src="./assets/report_pdf.png" alt="PDF Report" width="600">
</div>

---

### 6. Report Mode — Assistant Analyzing the Active Report

> The **ReportAgent** accesses the on-screen analysis directly. The user asks about the current report, and the agent extracts metrics, BTM events, and conclusions from the analysis without leaving the view.

<div align="center">
<img src="./assets/analysis_chat.png" alt="Analysis with Chat Assistant" width="800">
</div>

---

### 7. RAG Mode — Chat with WiFi Technical Documentation

> The **RAGAgent** answers using the internal knowledge base. It searches Qdrant for the most relevant technical documentation chunks and uses them as context to generate a grounded response. On the left, the Observability Dashboard displays real-time Langfuse traces for each LLM call: operation, model used, latency, and cost in USD per call.

<div align="center">
<img src="./assets/observability.png" alt="Observability" width="800">
</div>

---

### 8. RAG Mode — Response with Wireshark Filters

> Another query in RAG Mode: the user asks for Wireshark filters to analyze reassociation frames during a roaming event. The agent shows its chain of thought before delivering the technical response with the exact filters.

<div align="center">
<img src="./assets/chat_filters.png" alt="Technical chat with Wireshark filters" width="800">
</div>

---

## ⚙️ How It Works — Under the Hood

### Processing a PCAP Capture

When the user uploads a `.pcap` or `.pcapng` file, the backend runs this pipeline:

```
PCAP
  │
  ├─ 1. WiresharkTool (tshark) — extracts 802.11 frames
  │      ├─ Beacon frames (AP info, SSID, channel, band)
  │      ├─ BTM Request / Response (Band Steering)
  │      ├─ Probe Request / Response
  │      ├─ Association / Reassociation
  │      └─ Deauthentication
  │
  ├─ 2. BTMAnalyzer — computes Band Steering metrics
  │      ├─ Band transition latency
  │      ├─ BTM acceptance rate
  │      ├─ Time distribution across 2.4 GHz vs 5 GHz
  │      ├─ Sticky client and slow roaming detection
  │      └─ KVR verification (802.11k / 802.11v / 802.11r)
  │
  ├─ 3. LLM Analysis — generates technical insights
  │      ├─ Evaluates 802.11k/v/r compliance
  │      ├─ Identifies issues and anomalies
  │      └─ Produces improvement recommendations
  │
  └─ 4. Persistence across 3 databases in parallel
         ├─ PostgreSQL — structured analysis + metadata
         ├─ Qdrant     — embeddings for future semantic search
         └─ FalkorDB   — topology graph (Client → AP, transitions)
```

### Multi-Agent System

The core of the AI is an **AgentDispatcher** that classifies each query before processing it. The flow is:

```
User query
  │
  ├─ SecurityTool       — validates against malicious injection
  ├─ IntentClassifier   — detects intent (simple / rag / report / graph)
  │
  └─ Routing to the specialized agent:
       ├─ SimpleAgent   → direct LLM call, no context (response < 2s)
       ├─ RAGAgent      → searches Qdrant, uses top-5 documents as context
       ├─ ReportAgent   → accesses the active analysis (PostgreSQL + embeddings)
       └─ GraphAgent    → queries the network topology in FalkorDB (Cypher)
```

Each agent has its own system prompt and set of Skills, exposed as tools through a custom **Model Context Protocol (MCP)** layer, which reduces hallucinations and optimizes token usage. Before a response is returned, a **Judge System** validates and scores agent outputs for accuracy and relevance, filtering out low-quality responses.

### LLM Model Routing

The system doesn't rely on a single model: it dynamically selects the provider based on the task:

| Tier       | Model                | Use Case                              |
| ---------- | --------------------- | -------------------------------------- |
| `routing`  | Groq · Llama-3.3-70b | Intent classification                  |
| `cheap`    | Groq · Llama-3.3-70b | General conversation                   |
| `standard` | Gemini 2.0 Flash     | Capture analysis                       |
| `quality`  | GPT-4o               | Complex, high-precision analysis       |
| `fallback` | GPT-4o-mini          | If the primary provider fails          |

If a provider fails, the system automatically switches to the next one without interrupting the flow.

### Response Streaming (SSE)

The chat uses **Server-Sent Events** to stream the response token by token. The frontend receives different event types:

```
data: {"type": "status",  "content": "Classifying intent..."}
data: {"type": "thought", "content": "Detected intent: RAG"}
data: {"type": "thought", "content": "Searching vector database..."}
data: {"type": "token",   "content": "Band "}
data: {"type": "token",   "content": "Steering "}
data: {"type": "done",    "content": ""}
```

This allows the agent's internal reasoning to be shown in real time, before the final response appears.

### Batch Analysis

For large projects, Pipe offers parallel processing:

1. The user uploads multiple `.pcap` files at once along with their metadata (SSID, MAC, project)
2. Each file is queued in **Redis**
3. A background `QueueWorker` processes the tasks in parallel
4. The user monitors progress in real time (`processed / total`)
5. Once finished, the user can download:
    - **Full ZIP** — original captures + individual PDFs + summary
    - **Reports ZIP** — PDFs only
    - **Executive PDF** — consolidated diagnostic for the entire session

### Graph Database (FalkorDB)

Each analysis feeds a network topology graph with the following structure:

```
Nodes:
  - Client  { mac, ip, vendor }
  - AP      { bssid, ssid, channel, band }
  - Vendor  { vendor_id, name }
  - Project { project_id, name }

Relations:
  - (Client)-[CONNECTED_TO { timestamp, signal_dBm }]→(AP)
  - (Client)-[TRANSITIONED_TO { from_band, to_band, latency_ms }]→(AP)
  - (AP)-[BELONGS_TO]→(Project)
  - (Project)-[MANAGED_BY]→(Vendor)
```

The `GraphAgent` can query this topology with questions like:

> _"Which clients have transitioned to the AP with BSSID d4:ab:82:39:44:e1?"_

### Security and Access Control

Authentication is managed by **Keycloak** with three roles:

| Role      | Permissions                                                              |
| --------- | -------------------------------------------------------------------------- |
| `viewer` | Read-only access to reports and analyses                                    |
| `editor` | Upload captures, generate analyses, export PDFs                             |
| `admin`  | All of the above + user, vendor, project, and observability management     |

Every request to the backend first passes through **KrakenD** (API Gateway), which validates the JWT with Keycloak and forwards the `X-User-Id`, `X-User-Name`, and `X-Roles` headers to the backend.

---

## 🏗️ Tech Stack

| Layer          | Technologies                                                             |
| -------------- | --------------------------------------------------------------------------- |
| **Frontend**       | React 18, Vite, TailwindCSS                                              |
| **Backend**        | FastAPI, Python 3.12, Uvicorn                                            |
| **AI / Agents**    | LiteLLM, LangChain, MCP (custom), Groq (Llama 3.3), Gemini 2.0, GPT-4o  |
| **WiFi Analysis**  | tshark / Wireshark, PyShark                                              |
| **Databases**      | PostgreSQL, Qdrant (vector), FalkorDB (graph), Redis                     |
| **Auth**           | Keycloak (JWT, roles: admin / editor / viewer)                          |
| **Gateway**        | KrakenD                                                                  |
| **Observability**  | Langfuse                                                                 |
| **Infra**          | Docker Compose, Nginx, AWS                                               |

---

## ⚙️ Architecture Overview

```
Browser (React)
    │
    ▼
KrakenD (API Gateway + JWT validation)
    │
    ▼
FastAPI Backend
    ├─ AgentDispatcher → SecurityTool → IntentClassifier
    │       ├─ SimpleAgent   (general conversation)
    │       ├─ RAGAgent      (Qdrant + technical documentation)
    │       ├─ ReportAgent   (analysis over existing reports)
    │       └─ GraphAgent    (network topology in FalkorDB)
    │
    ├─ BandSteeringService → tshark → BTMAnalyzer → LLM → PDF
    └─ MassiveAnalysisQueue (Redis + background workers)
         │
    ┌────┴───────────────────────────────────────┐
    │  PostgreSQL · Qdrant · Redis · FalkorDB    │
    └────────────────────────────────────────────┘
              ↕ Langfuse (traces + costs + latencies)
```

---

<div align="center">
<p>Developed by <a href="https://github.com/miguelczz"><b>miguelczz</b></a> · 2026</p>
</div>

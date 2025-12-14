# Multi-Agent Email System Architecture

## Document Information

| Attribute | Value |
|-----------|-------|
| Version | 1.0.0 |
| Status | Draft |
| Created | 2025-12-14 |
| Authors | Architecture Team |

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [System Overview](#2-system-overview)
3. [Architecture Principles](#3-architecture-principles)
4. [System Components](#4-system-components)
5. [Technology Stack](#5-technology-stack)
6. [Communication Patterns](#6-communication-patterns)
7. [Data Flow](#7-data-flow)
8. [Execution Flow](#8-execution-flow)
9. [API Contracts](#9-api-contracts)
10. [State Management](#10-state-management)
11. [Error Handling Strategy](#11-error-handling-strategy)
12. [Logging and Observability](#12-logging-and-observability)
13. [Directory Structure](#13-directory-structure)
14. [Deployment Model](#14-deployment-model)
15. [Design Decisions](#15-design-decisions)

---

## 1. Executive Summary

This document describes the architecture for a multi-agent email system built using Google's Agent-to-Agent (A2A) protocol and LangGraph. The system enables a Supervisor Agent to orchestrate email-sending tasks by delegating to a specialized Mail Agent.

### Core Capability

```
User provides instructions file → Supervisor parses intent → Mail Agent drafts & sends email
```

### Key Design Choices

- **A2A Protocol**: Industry-standard agent interoperability (Google-backed, Linux Foundation governed)
- **LangGraph**: Graph-based state machine for supervisor orchestration
- **Gemini 2.5 Flash**: LLM for intent parsing and email drafting
- **aiosmtpd**: Full SMTP mock server for testing

---

## 2. System Overview

### 2.1 System Context Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                    SYSTEM BOUNDARY                                   │
│                                                                                      │
│  ┌─────────────┐                                                                    │
│  │             │                                                                    │
│  │  END USER   │                                                                    │
│  │             │                                                                    │
│  └──────┬──────┘                                                                    │
│         │                                                                           │
│         │ HTTP POST /process                                                        │
│         │ {instructions_file: "/path/to/file.txt"}                                  │
│         ▼                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐               │
│  │                    SUPERVISOR AGENT                              │               │
│  │                    (Port 8000)                                   │               │
│  │                                                                  │               │
│  │   ┌──────────────────────────────────────────────────────────┐  │               │
│  │   │                   LangGraph StateGraph                    │  │               │
│  │   │                                                           │  │               │
│  │   │  [READ_FILE] → [PARSE_INTENT] → [DISCOVER] → [SUBMIT]    │  │               │
│  │   │                                                           │  │               │
│  │   └──────────────────────────────────────────────────────────┘  │               │
│  │                                                                  │               │
│  └─────────────────────────────┬────────────────────────────────────┘               │
│                                │                                                    │
│                                │ A2A Protocol (JSON-RPC 2.0 over HTTP)              │
│                                │                                                    │
│                                ▼                                                    │
│  ┌─────────────────────────────────────────────────────────────────┐               │
│  │                      MAIL AGENT                                  │               │
│  │                      (Port 8001)                                 │               │
│  │                                                                  │               │
│  │   ┌──────────────────────────────────────────────────────────┐  │               │
│  │   │  A2A Server + Gemini Email Drafter + SMTP Client         │  │               │
│  │   └──────────────────────────────────────────────────────────┘  │               │
│  │                                                                  │               │
│  └─────────────────────────────┬────────────────────────────────────┘               │
│                                │                                                    │
│                                │ SMTP Protocol                                      │
│                                │                                                    │
│                                ▼                                                    │
│  ┌─────────────────────────────────────────────────────────────────┐               │
│  │                   MOCK MAIL SERVER                               │               │
│  │                   (SMTP: 8025, REST: 8002)                       │               │
│  │                                                                  │               │
│  │   ┌────────────────────┐    ┌─────────────────────────────────┐ │               │
│  │   │  aiosmtpd Handler  │    │  REST API (View Captured Emails)│ │               │
│  │   └────────────────────┘    └─────────────────────────────────┘ │               │
│  │                                                                  │               │
│  └──────────────────────────────────────────────────────────────────┘               │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Component Responsibilities

| Component | Primary Responsibility | Secondary Responsibilities |
|-----------|------------------------|---------------------------|
| **Supervisor Agent** | Orchestrate email tasks | Parse instructions, Route to agents, Handle retries |
| **Mail Agent** | Draft and send emails | Expose A2A interface, Use LLM for drafting |
| **Mock Mail Server** | Capture sent emails | Provide REST API for verification |

---

## 3. Architecture Principles

### 3.1 Core Principles

1. **Separation of Concerns**: Each agent has a single, well-defined responsibility
2. **Protocol-First**: A2A protocol ensures interoperability and future extensibility
3. **Fail-Fast with Retry**: Detect failures early, retry with exponential backoff
4. **Observability by Default**: Structured logging with correlation IDs throughout
5. **No Fallbacks/Defaults**: Raise exceptions on missing configuration

### 3.2 Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Response Time | < 30 seconds | LLM calls + SMTP operations |
| Retry Attempts | 3 | Balance between resilience and failure detection |
| Log Format | JSON | Machine-parseable, correlation-friendly |
| Max File Lines | 800 per file | Maintainability constraint |

---

## 4. System Components

### 4.1 Supervisor Agent

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           SUPERVISOR AGENT                                   │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                         FastAPI Server (Port 8000)                      │ │
│  │                                                                         │ │
│  │   Endpoints:                                                            │ │
│  │     POST /process  - Main entry point                                   │ │
│  │     GET  /health   - Health check                                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                       │
│                                      ▼                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                         LangGraph StateGraph                            │ │
│  │                                                                         │ │
│  │   State Schema:                                                         │ │
│  │     - messages: list[Message]                                           │ │
│  │     - instructions_file: str                                            │ │
│  │     - raw_instructions: str                                             │ │
│  │     - parsed_intent: dict                                               │ │
│  │     - mail_agent_card: dict                                             │ │
│  │     - task_id: str                                                      │ │
│  │     - task_status: str                                                  │ │
│  │     - retry_count: int                                                  │ │
│  │     - result: dict                                                      │ │
│  │     - error: Optional[str]                                              │ │
│  │                                                                         │ │
│  │   Nodes:                                                                │ │
│  │     [READ_FILE] → [PARSE_INTENT] → [DISCOVER_AGENT] → [SUBMIT_TASK]    │ │
│  │                                                                         │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                       │
│                                      ▼                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                           A2A Client                                    │ │
│  │                                                                         │ │
│  │   - Discovers agents via /.well-known/agent-card.json                   │ │
│  │   - Submits tasks via JSON-RPC                                          │ │
│  │   - Handles retries with exponential backoff                            │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 4.1.1 LangGraph Node Responsibilities

| Node | Responsibility | Input | Output |
|------|----------------|-------|--------|
| `READ_FILE` | Read instructions from file path | `instructions_file` | `raw_instructions` |
| `PARSE_INTENT` | Use Gemini to extract structured intent | `raw_instructions` | `parsed_intent` |
| `DISCOVER_AGENT` | Fetch Mail Agent's A2A card | - | `mail_agent_card` |
| `SUBMIT_TASK` | Submit task to Mail Agent via A2A | `parsed_intent` | `task_id`, `task_status`, `result` |

### 4.2 Mail Agent

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              MAIL AGENT                                      │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                      A2A Server (Port 8001)                             │ │
│  │                                                                         │ │
│  │   Endpoints:                                                            │ │
│  │     GET  /.well-known/agent-card.json  - Agent discovery                │ │
│  │     POST /a2a                          - JSON-RPC task handling         │ │
│  │     GET  /health                       - Health check                   │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                       │
│                                      ▼                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                        Task Handler                                     │ │
│  │                                                                         │ │
│  │   Supported Skills:                                                     │ │
│  │     - draft-and-send-email: Draft professional email and send via SMTP │ │
│  │                                                                         │ │
│  │   Task Lifecycle:                                                       │ │
│  │     submitted → working → completed/failed                              │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                       │
│                                      ▼                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                       Email Drafter (Gemini)                            │ │
│  │                                                                         │ │
│  │   - Receives request details (recipient, content type, quantity)        │ │
│  │   - Generates professional email using Gemini 2.5 Flash                 │ │
│  │   - Returns structured email (subject, body)                            │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                       │
│                                      ▼                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                         SMTP Client                                     │ │
│  │                                                                         │ │
│  │   - Connects to Mock Mail Server (localhost:8025)                       │ │
│  │   - Sends email via aiosmtplib                                          │ │
│  │   - Returns message ID on success                                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 Mock Mail Server

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           MOCK MAIL SERVER                                   │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                      SMTP Handler (Port 8025)                           │ │
│  │                                                                         │ │
│  │   - aiosmtpd-based SMTP server                                          │ │
│  │   - Accepts all incoming mail                                           │ │
│  │   - Stores in in-memory storage                                         │ │
│  │   - Assigns unique message ID                                           │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                       │
│                                      ▼                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                      In-Memory Storage                                  │ │
│  │                                                                         │ │
│  │   Data Structure:                                                       │ │
│  │     emails: Dict[str, EmailRecord]                                      │ │
│  │                                                                         │ │
│  │   EmailRecord:                                                          │ │
│  │     - id: str (UUID)                                                    │ │
│  │     - from_addr: str                                                    │ │
│  │     - to_addrs: List[str]                                               │ │
│  │     - subject: str                                                      │ │
│  │     - body: str                                                         │ │
│  │     - headers: Dict[str, str]                                           │ │
│  │     - timestamp: datetime                                               │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                       │
│                                      ▼                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                      REST API (Port 8002)                               │ │
│  │                                                                         │ │
│  │   Endpoints:                                                            │ │
│  │     GET    /emails      - List all captured emails                      │ │
│  │     GET    /emails/{id} - Get specific email                            │ │
│  │     DELETE /emails      - Clear all emails                              │ │
│  │     GET    /health      - Health check                                  │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Technology Stack

### 5.1 Component Dependencies

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DEPENDENCY GRAPH                                   │
└─────────────────────────────────────────────────────────────────────────────┘

Supervisor Agent
    │
    ├── langgraph              (State machine orchestration)
    ├── langchain-google-genai (Gemini 2.5 Flash integration)
    ├── a2a-python             (A2A protocol client)
    ├── httpx                  (Async HTTP client)
    ├── fastapi                (REST API framework)
    ├── uvicorn                (ASGI server)
    └── python-json-logger     (Structured logging)

Mail Agent
    │
    ├── a2a-python             (A2A protocol server)
    ├── langchain-google-genai (Gemini 2.5 Flash integration)
    ├── aiosmtplib             (Async SMTP client)
    ├── fastapi                (REST API framework)
    ├── uvicorn                (ASGI server)
    └── python-json-logger     (Structured logging)

Mock Mail Server
    │
    ├── aiosmtpd               (SMTP server)
    ├── fastapi                (REST API framework)
    ├── uvicorn                (ASGI server)
    └── python-json-logger     (Structured logging)
```

### 5.2 Technology Matrix

| Layer | Technology | Version | Purpose |
|-------|------------|---------|---------|
| **Orchestration** | LangGraph | >= 0.3.0 | State machine for supervisor |
| **LLM** | Gemini 2.5 Flash | - | Intent parsing, email drafting |
| **Agent Protocol** | A2A (a2a-python) | >= 0.3.0 | Agent interoperability |
| **HTTP Framework** | FastAPI | >= 0.100 | REST APIs |
| **HTTP Client** | httpx | >= 0.25 | Async HTTP calls |
| **SMTP Client** | aiosmtplib | >= 3.0 | Async SMTP sending |
| **SMTP Server** | aiosmtpd | >= 1.4 | Mock mail server |
| **Logging** | python-json-logger | >= 2.0 | Structured JSON logs |
| **Runtime** | Python | >= 3.11 | Language runtime |

---

## 6. Communication Patterns

### 6.1 Protocol Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        COMMUNICATION PROTOCOLS                               │
└─────────────────────────────────────────────────────────────────────────────┘

User ──────────────────────────────────────────────────────── Supervisor
         │
         │  HTTP/REST
         │  Content-Type: application/json
         │
         └─────────────────────────────────────────────────────────────────────

Supervisor ────────────────────────────────────────────────── Mail Agent
              │
              │  A2A Protocol (JSON-RPC 2.0 over HTTP)
              │  Discovery: GET /.well-known/agent-card.json
              │  Tasks: POST /a2a
              │
              └─────────────────────────────────────────────────────────────────

Mail Agent ────────────────────────────────────────────────── Mock SMTP
              │
              │  SMTP Protocol (RFC 5321)
              │  Port: 8025
              │  AUTH: None (mock server)
              │
              └─────────────────────────────────────────────────────────────────
```

### 6.2 A2A Protocol Details

#### 6.2.1 Agent Discovery

```
Sequence: Agent Discovery

Supervisor                                   Mail Agent
    │                                            │
    │  GET /.well-known/agent-card.json          │
    │ ──────────────────────────────────────────▶│
    │                                            │
    │  200 OK                                    │
    │  {                                         │
    │    "name": "Mail Agent",                   │
    │    "url": "http://localhost:8001/a2a",     │
    │    "skills": [                             │
    │      {                                     │
    │        "id": "draft-and-send-email",       │
    │        "name": "Draft and Send Email"      │
    │      }                                     │
    │    ]                                       │
    │  }                                         │
    │ ◀──────────────────────────────────────────│
    │                                            │
```

#### 6.2.2 Task Submission

```
Sequence: Task Submission via A2A

Supervisor                                   Mail Agent
    │                                            │
    │  POST /a2a                                 │
    │  {                                         │
    │    "jsonrpc": "2.0",                       │
    │    "method": "tasks/submit",               │
    │    "params": {                             │
    │      "skillId": "draft-and-send-email",    │
    │      "messages": [...]                     │
    │    },                                      │
    │    "id": "req-001"                         │
    │  }                                         │
    │ ──────────────────────────────────────────▶│
    │                                            │
    │                                            │ [Process Task]
    │                                            │ [Draft Email via Gemini]
    │                                            │ [Send via SMTP]
    │                                            │
    │  {                                         │
    │    "jsonrpc": "2.0",                       │
    │    "result": {                             │
    │      "id": "task-uuid",                    │
    │      "status": {"state": "completed"},     │
    │      "artifacts": [...]                    │
    │    },                                      │
    │    "id": "req-001"                         │
    │  }                                         │
    │ ◀──────────────────────────────────────────│
    │                                            │
```

### 6.3 A2A Task States

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         A2A TASK STATE MACHINE                               │
└─────────────────────────────────────────────────────────────────────────────┘

                              ┌───────────┐
                              │ submitted │
                              └─────┬─────┘
                                    │
                                    ▼
                              ┌───────────┐
                              │  working  │
                              └─────┬─────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
                    ▼               ▼               ▼
              ┌───────────┐  ┌───────────┐  ┌───────────┐
              │ completed │  │  failed   │  │ canceled  │
              └───────────┘  └───────────┘  └───────────┘
                   │               │               │
                   └───────────────┴───────────────┘
                                   │
                                   ▼
                            [Terminal States]
```

---

## 7. Data Flow

### 7.1 End-to-End Data Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              DATA FLOW                                       │
└─────────────────────────────────────────────────────────────────────────────┘

Step 1: User Input
──────────────────
User provides: {"instructions_file": "/path/to/instructions.txt"}

Step 2: File Content
────────────────────
instructions.txt contains:
  "Send mail to raj@gmail.com and ask for an excel file having 10 rows of food recipes"

Step 3: Parsed Intent (Gemini Output)
─────────────────────────────────────
{
  "action": "send_email",
  "recipient": "raj@gmail.com",
  "request_type": "data_request",
  "request_details": {
    "format": "excel",
    "content": "food recipes",
    "quantity": 10
  }
}

Step 4: A2A Task Message
────────────────────────
{
  "to": "raj@gmail.com",
  "request_type": "data_request",
  "request_details": {
    "format": "excel",
    "content": "food recipes",
    "quantity": 10
  }
}

Step 5: Drafted Email (Gemini Output)
─────────────────────────────────────
Subject: Request for Food Recipes Data

Dear Raj,

I hope this email finds you well.

I am writing to kindly request your assistance in providing
an Excel file containing 10 food recipes. If possible, please
include details such as recipe names, ingredients, and
preparation instructions.

Please feel free to reach out if you have any questions.

Thank you for your time and assistance.

Best regards,
Automated Mail Agent

Step 6: SMTP Message
────────────────────
FROM: mail-agent@system.local
TO: raj@gmail.com
SUBJECT: Request for Food Recipes Data
BODY: [Above email content]

Step 7: Final Response
──────────────────────
{
  "status": "completed",
  "result": "Email successfully sent to raj@gmail.com",
  "details": {
    "recipient": "raj@gmail.com",
    "message_id": "msg-uuid-001"
  }
}
```

### 7.2 Data Transformation Pipeline

```
┌───────────────┐    ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│ Raw Text File │───▶│ Parsed Intent │───▶│ A2A Message   │───▶│ SMTP Email    │
│               │    │   (JSON)      │    │   (JSON-RPC)  │    │   (RFC 5322)  │
└───────────────┘    └───────────────┘    └───────────────┘    └───────────────┘
        │                    │                    │                    │
        │                    │                    │                    │
   File System          Gemini LLM          A2A Protocol         SMTP Protocol
```

---

## 8. Execution Flow

### 8.1 Complete Sequence Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           COMPLETE EXECUTION SEQUENCE                                │
└─────────────────────────────────────────────────────────────────────────────────────┘

User          Supervisor          Gemini          Mail Agent          Mock SMTP
  │                │                 │                 │                   │
  │  POST /process │                 │                 │                   │
  │  {file: "..."}│                 │                 │                   │
  │───────────────▶│                 │                 │                   │
  │                │                 │                 │                   │
  │                │ [READ_FILE]     │                 │                   │
  │                │ Read file       │                 │                   │
  │                │                 │                 │                   │
  │                │ [PARSE_INTENT]  │                 │                   │
  │                │ Parse this text │                 │                   │
  │                │────────────────▶│                 │                   │
  │                │                 │                 │                   │
  │                │ {parsed_intent} │                 │                   │
  │                │◀────────────────│                 │                   │
  │                │                 │                 │                   │
  │                │ [DISCOVER_AGENT]│                 │                   │
  │                │ GET /agent-card │                 │                   │
  │                │─────────────────────────────────▶│                   │
  │                │                 │                 │                   │
  │                │ {agent_card}    │                 │                   │
  │                │◀─────────────────────────────────│                   │
  │                │                 │                 │                   │
  │                │ [SUBMIT_TASK]   │                 │                   │
  │                │ POST /a2a       │                 │                   │
  │                │─────────────────────────────────▶│                   │
  │                │                 │                 │                   │
  │                │                 │                 │ Draft email       │
  │                │                 │◀────────────────│                   │
  │                │                 │                 │                   │
  │                │                 │ {email_content} │                   │
  │                │                 │────────────────▶│                   │
  │                │                 │                 │                   │
  │                │                 │                 │ SMTP SEND         │
  │                │                 │                 │──────────────────▶│
  │                │                 │                 │                   │
  │                │                 │                 │ 250 OK            │
  │                │                 │                 │◀──────────────────│
  │                │                 │                 │                   │
  │                │ {task: completed}                 │                   │
  │                │◀─────────────────────────────────│                   │
  │                │                 │                 │                   │
  │ {status: ok}   │                 │                 │                   │
  │◀───────────────│                 │                 │                   │
  │                │                 │                 │                   │
```

### 8.2 LangGraph State Transitions

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     LANGGRAPH STATE TRANSITION DIAGRAM                       │
└─────────────────────────────────────────────────────────────────────────────┘

                              ┌─────────┐
                              │  START  │
                              └────┬────┘
                                   │
                                   ▼
                         ┌─────────────────┐
                         │   READ_FILE     │
                         │                 │
                         │ State Updates:  │
                         │ + raw_instruct- │
                         │   ions          │
                         └────────┬────────┘
                                  │
                                  ▼
                         ┌─────────────────┐
                         │  PARSE_INTENT   │
                         │                 │
                         │ State Updates:  │
                         │ + parsed_intent │
                         └────────┬────────┘
                                  │
                                  ▼
                         ┌─────────────────┐
                         │ DISCOVER_AGENT  │
                         │                 │
                         │ State Updates:  │
                         │ + mail_agent_   │
                         │   card          │
                         └────────┬────────┘
                                  │
                                  ▼
                         ┌─────────────────┐
                    ┌───▶│  SUBMIT_TASK    │───────┐
                    │    │                 │       │
                    │    │ State Updates:  │       │
              Retry │    │ + task_id       │       │ Success
        (retry < 3) │    │ + task_status   │       │
                    │    │ + retry_count   │       │
                    │    └────────┬────────┘       │
                    │             │                │
                    │             │ Failure        │
                    │             ▼                ▼
                    │    ┌─────────────────┐  ┌────────┐
                    └────│     RETRY       │  │ FINAL- │
                         │   (backoff)     │  │  IZE   │
                         └─────────────────┘  └────┬───┘
                                                   │
                                                   ▼
                                              ┌────────┐
                                              │  END   │
                                              └────────┘
```

---

## 9. API Contracts

### 9.1 Supervisor Agent API

```yaml
# ═══════════════════════════════════════════════════════════════════════════
#                        SUPERVISOR AGENT API
#                           Port: 8000
# ═══════════════════════════════════════════════════════════════════════════

openapi: 3.0.0
info:
  title: Supervisor Agent API
  version: 1.0.0
  description: Orchestrates email tasks by delegating to Mail Agent

paths:
  /process:
    post:
      summary: Process instructions file and execute email task
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - instructions_file
              properties:
                instructions_file:
                  type: string
                  description: Absolute path to instructions file
                  example: "/path/to/instructions.txt"
      responses:
        '200':
          description: Task completed successfully
          content:
            application/json:
              schema:
                type: object
                properties:
                  status:
                    type: string
                    enum: [completed]
                  task_id:
                    type: string
                    format: uuid
                  result:
                    type: string
                  details:
                    type: object
                    properties:
                      recipient:
                        type: string
                      message_id:
                        type: string
        '500':
          description: Task failed after retries
          content:
            application/json:
              schema:
                type: object
                properties:
                  status:
                    type: string
                    enum: [failed]
                  task_id:
                    type: string
                  error:
                    type: string
                  last_error:
                    type: string

  /health:
    get:
      summary: Health check endpoint
      responses:
        '200':
          description: Service is healthy
          content:
            application/json:
              schema:
                type: object
                properties:
                  status:
                    type: string
                    enum: [healthy]
                  agents:
                    type: object
                    properties:
                      mail_agent:
                        type: string
                        enum: [connected, disconnected]
```

### 9.2 Mail Agent API (A2A)

```yaml
# ═══════════════════════════════════════════════════════════════════════════
#                           MAIL AGENT A2A API
#                              Port: 8001
# ═══════════════════════════════════════════════════════════════════════════

# Agent Card Schema
# Endpoint: GET /.well-known/agent-card.json

AgentCard:
  name: "Mail Agent"
  description: "Drafts and sends email requests to recipients"
  version: "1.0.0"
  url: "http://localhost:8001/a2a"
  capabilities:
    streaming: false
    pushNotifications: false
  defaultInputModes:
    - "application/json"
  defaultOutputModes:
    - "application/json"
  skills:
    - id: "draft-and-send-email"
      name: "Draft and Send Email"
      description: "Uses LLM to draft professional email and sends via SMTP"
      inputModes:
        - "application/json"
      outputModes:
        - "application/json"
      examples:
        - input:
            to: "user@example.com"
            request_type: "data_request"
            request_details:
              format: "excel"
              content: "food recipes"
              quantity: 10
          output:
            success: true
            message_id: "msg-123"

# ═══════════════════════════════════════════════════════════════════════════

# Task Submission (JSON-RPC)
# Endpoint: POST /a2a

TaskSubmitRequest:
  jsonrpc: "2.0"
  method: "tasks/submit"
  params:
    skillId: "draft-and-send-email"
    messages:
      - role: "user"
        parts:
          - contentType: "application/json"
            data:
              to: "raj@gmail.com"
              request_type: "data_request"
              request_details:
                format: "excel"
                content: "food recipes"
                quantity: 10
  id: "req-001"

TaskSubmitResponse_Success:
  jsonrpc: "2.0"
  result:
    id: "task-uuid"
    contextId: "ctx-uuid"
    status:
      state: "completed"
      timestamp: "2025-12-14T10:30:00Z"
    artifacts:
      - id: "email-result"
        parts:
          - contentType: "application/json"
            data:
              success: true
              message_id: "msg-uuid"
              recipient: "raj@gmail.com"
              subject: "Request for Food Recipes Data"
  id: "req-001"

TaskSubmitResponse_Failure:
  jsonrpc: "2.0"
  error:
    code: -32000
    message: "SMTP connection failed"
    data:
      retry_count: 3
  id: "req-001"
```

### 9.3 Mock Mail Server API

```yaml
# ═══════════════════════════════════════════════════════════════════════════
#                        MOCK MAIL SERVER API
#                     SMTP: 8025, REST: 8002
# ═══════════════════════════════════════════════════════════════════════════

openapi: 3.0.0
info:
  title: Mock Mail Server REST API
  version: 1.0.0
  description: REST API to view captured emails from SMTP server

paths:
  /emails:
    get:
      summary: List all captured emails
      responses:
        '200':
          description: List of emails
          content:
            application/json:
              schema:
                type: object
                properties:
                  count:
                    type: integer
                  emails:
                    type: array
                    items:
                      $ref: '#/components/schemas/EmailSummary'
    delete:
      summary: Clear all captured emails
      responses:
        '200':
          description: Emails cleared
          content:
            application/json:
              schema:
                type: object
                properties:
                  deleted:
                    type: integer

  /emails/{id}:
    get:
      summary: Get specific email by ID
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: Email details
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/EmailDetail'
        '404':
          description: Email not found

  /health:
    get:
      summary: Health check
      responses:
        '200':
          description: Service is healthy
          content:
            application/json:
              schema:
                type: object
                properties:
                  status:
                    type: string
                  smtp_port:
                    type: integer
                  emails_captured:
                    type: integer

components:
  schemas:
    EmailSummary:
      type: object
      properties:
        id:
          type: string
        from:
          type: string
        to:
          type: array
          items:
            type: string
        subject:
          type: string
        timestamp:
          type: string
          format: date-time

    EmailDetail:
      type: object
      properties:
        id:
          type: string
        from:
          type: string
        to:
          type: array
          items:
            type: string
        subject:
          type: string
        body:
          type: string
        headers:
          type: object
          additionalProperties:
            type: string
        timestamp:
          type: string
          format: date-time
```

---

## 10. State Management

### 10.1 Supervisor State Schema

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      SUPERVISOR STATE SCHEMA                                 │
└─────────────────────────────────────────────────────────────────────────────┘

SupervisorState (TypedDict):
│
├── messages: Annotated[list[Message], add_messages]
│   │
│   └── Accumulates conversation history across nodes
│
├── instructions_file: str
│   │
│   └── Input: absolute path to instructions file
│
├── raw_instructions: str
│   │
│   └── Output of READ_FILE: raw text content
│
├── parsed_intent: dict
│   │
│   ├── action: str                    # "send_email"
│   ├── recipient: str                 # "raj@gmail.com"
│   ├── request_type: str              # "data_request"
│   └── request_details: dict
│       ├── format: str                # "excel"
│       ├── content: str               # "food recipes"
│       └── quantity: int              # 10
│
├── mail_agent_card: dict
│   │
│   └── Cached A2A agent card from discovery
│
├── task_id: str
│   │
│   └── A2A task ID returned from Mail Agent
│
├── task_status: str
│   │
│   └── One of: submitted, working, completed, failed
│
├── retry_count: int
│   │
│   └── Current retry attempt (0-3)
│
├── result: dict
│   │
│   ├── success: bool
│   ├── message_id: str
│   └── recipient: str
│
└── error: Optional[str]
    │
    └── Error message if task failed
```

### 10.2 State Transitions by Node

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      STATE TRANSITIONS BY NODE                               │
└─────────────────────────────────────────────────────────────────────────────┘

Node: READ_FILE
───────────────
Before: {instructions_file: "/path/to/file.txt"}
After:  {instructions_file: "/path/to/file.txt",
         raw_instructions: "Send mail to raj@gmail.com..."}

Node: PARSE_INTENT
──────────────────
Before: {raw_instructions: "Send mail to raj@gmail.com..."}
After:  {raw_instructions: "...",
         parsed_intent: {action: "send_email", recipient: "raj@gmail.com", ...}}

Node: DISCOVER_AGENT
────────────────────
Before: {parsed_intent: {...}}
After:  {parsed_intent: {...},
         mail_agent_card: {name: "Mail Agent", url: "...", skills: [...]}}

Node: SUBMIT_TASK (Success)
───────────────────────────
Before: {parsed_intent: {...}, mail_agent_card: {...}}
After:  {parsed_intent: {...}, mail_agent_card: {...},
         task_id: "uuid", task_status: "completed",
         result: {success: true, message_id: "..."}}

Node: SUBMIT_TASK (Failure with retry)
──────────────────────────────────────
Before: {retry_count: 0, ...}
After:  {retry_count: 1, task_status: "failed", error: "SMTP timeout"}
        [Transitions back to SUBMIT_TASK after backoff]

Node: SUBMIT_TASK (Final failure)
─────────────────────────────────
Before: {retry_count: 3, ...}
After:  {retry_count: 3, task_status: "failed",
         error: "Failed after 3 retries: SMTP timeout"}
        [Transitions to END]
```

---

## 11. Error Handling Strategy

### 11.1 Retry with Exponential Backoff

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     EXPONENTIAL BACKOFF ALGORITHM                            │
└─────────────────────────────────────────────────────────────────────────────┘

Configuration:
  MAX_RETRIES     = 3
  INITIAL_DELAY   = 1 second
  MULTIPLIER      = 2
  MAX_DELAY       = 10 seconds

Algorithm:
──────────
  delay = INITIAL_DELAY

  for attempt in 1 to MAX_RETRIES:
      try:
          result = execute_task()
          return result
      except RetryableError as e:
          if attempt == MAX_RETRIES:
              notify_supervisor(error=e)
              raise FinalFailureError(e)

          log.warning(f"Attempt {attempt} failed, retrying in {delay}s")
          sleep(delay)
          delay = min(delay * MULTIPLIER, MAX_DELAY)

Timeline:
─────────
  Attempt 1 ─────── Fail ─── Wait 1s ───┐
                                        │
  Attempt 2 ◀───────────────────────────┘
            ─────── Fail ─── Wait 2s ───┐
                                        │
  Attempt 3 ◀───────────────────────────┘
            ─────── Fail ─── FINAL FAILURE
                             Notify Supervisor
```

### 11.2 Error Categories

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           ERROR CATEGORIES                                   │
└─────────────────────────────────────────────────────────────────────────────┘

Retryable Errors (will retry with backoff):
───────────────────────────────────────────
  - SMTP connection timeout
  - SMTP connection refused
  - A2A agent temporarily unavailable
  - Network errors (transient)
  - LLM rate limiting

Non-Retryable Errors (fail immediately):
────────────────────────────────────────
  - File not found
  - Invalid file format
  - Invalid email address
  - A2A agent card not found (404)
  - Authentication failure
  - Validation errors

Error Response Format:
──────────────────────
{
  "status": "failed",
  "task_id": "uuid",
  "error": "Human-readable error message",
  "error_code": "SMTP_CONNECTION_FAILED",
  "retry_count": 3,
  "last_error": "Connection refused to localhost:8025"
}
```

### 11.3 Supervisor Notification

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SUPERVISOR NOTIFICATION ON FAILURE                        │
└─────────────────────────────────────────────────────────────────────────────┘

When: After all retry attempts exhausted

Mechanism: State update in LangGraph

State Update:
─────────────
{
  "task_status": "failed",
  "error": "Failed to send email after 3 retries: SMTP connection refused",
  "retry_count": 3
}

Logged Event:
─────────────
{
  "timestamp": "2025-12-14T10:35:00Z",
  "level": "ERROR",
  "service": "supervisor",
  "correlation_id": "req-001",
  "node": "SUBMIT_TASK",
  "message": "Task failed after max retries",
  "data": {
    "task_id": "task-uuid",
    "retry_count": 3,
    "last_error": "SMTP connection refused"
  }
}
```

---

## 12. Logging and Observability

### 12.1 Structured Logging Format

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      STRUCTURED LOGGING FORMAT                               │
└─────────────────────────────────────────────────────────────────────────────┘

Log Schema (JSON):
──────────────────
{
  "timestamp": "ISO8601 datetime",
  "level": "DEBUG|INFO|WARNING|ERROR",
  "service": "supervisor|mail_agent|mock_mail_server",
  "correlation_id": "Request correlation ID (propagated across services)",
  "node": "LangGraph node name (supervisor only)",
  "message": "Human-readable message",
  "data": {
    // Structured context-specific data
  }
}

Example Logs:
─────────────

# Supervisor: File read
{"timestamp": "2025-12-14T10:30:00Z", "level": "INFO", "service": "supervisor",
 "correlation_id": "req-001", "node": "READ_FILE",
 "message": "Instructions file read successfully",
 "data": {"file": "/path/to/instructions.txt", "length": 85}}

# Supervisor: Intent parsed
{"timestamp": "2025-12-14T10:30:01Z", "level": "INFO", "service": "supervisor",
 "correlation_id": "req-001", "node": "PARSE_INTENT",
 "message": "Intent parsed successfully",
 "data": {"action": "send_email", "recipient": "raj@gmail.com"}}

# Mail Agent: Email drafted
{"timestamp": "2025-12-14T10:30:02Z", "level": "INFO", "service": "mail_agent",
 "correlation_id": "req-001",
 "message": "Email drafted using Gemini",
 "data": {"recipient": "raj@gmail.com", "subject": "Request for Food Recipes Data"}}

# Mail Agent: SMTP sent
{"timestamp": "2025-12-14T10:30:03Z", "level": "INFO", "service": "mail_agent",
 "correlation_id": "req-001",
 "message": "Email sent via SMTP",
 "data": {"message_id": "msg-001", "smtp_response": "250 OK"}}

# Mock Mail Server: Email captured
{"timestamp": "2025-12-14T10:30:03Z", "level": "INFO", "service": "mock_mail_server",
 "correlation_id": "req-001",
 "message": "Email captured",
 "data": {"id": "msg-001", "from": "mail-agent@system.local", "to": "raj@gmail.com"}}
```

### 12.2 Correlation ID Propagation

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CORRELATION ID PROPAGATION                                │
└─────────────────────────────────────────────────────────────────────────────┘

Flow:
─────

User Request
    │
    │  HTTP Header: X-Correlation-ID: req-001
    │  (Generated if not provided)
    ▼
┌─────────────────┐
│   Supervisor    │  correlation_id = "req-001"
│                 │  All logs include this ID
└────────┬────────┘
         │
         │  A2A Request Header: X-Correlation-ID: req-001
         ▼
┌─────────────────┐
│   Mail Agent    │  correlation_id = "req-001" (extracted from header)
│                 │  All logs include this ID
└────────┬────────┘
         │
         │  SMTP: X-Correlation-ID header in email
         ▼
┌─────────────────┐
│ Mock Mail Server│  correlation_id = "req-001" (extracted from email header)
│                 │  All logs include this ID
└─────────────────┘

Result: All logs across all services can be traced via correlation_id
```

### 12.3 Log Levels by Event Type

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        LOG LEVELS BY EVENT TYPE                              │
└─────────────────────────────────────────────────────────────────────────────┘

DEBUG:
  - Raw file content (truncated)
  - Full LLM prompts and responses
  - A2A request/response bodies
  - SMTP protocol details

INFO:
  - Request received
  - Node transitions
  - Task status changes
  - Email sent successfully
  - Email captured

WARNING:
  - Retry attempt started
  - Slow response from LLM
  - Near timeout conditions

ERROR:
  - File not found
  - LLM call failed
  - SMTP connection failed
  - A2A agent unavailable
  - Max retries exceeded
```

---

## 13. Directory Structure

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PROJECT DIRECTORY STRUCTURE                          │
└─────────────────────────────────────────────────────────────────────────────┘

/workspaces/infor-agent-2/
│
├── src/
│   └── infor_agent/
│       │
│       ├── __init__.py
│       │
│       ├── supervisor/
│       │   ├── __init__.py
│       │   ├── agent.py                 # LangGraph supervisor graph
│       │   ├── state.py                 # SupervisorState TypedDict
│       │   ├── server.py                # FastAPI server (port 8000)
│       │   └── nodes/
│       │       ├── __init__.py
│       │       ├── read_file.py         # READ_FILE node
│       │       ├── parse_intent.py      # PARSE_INTENT node
│       │       ├── discover_agent.py    # DISCOVER_AGENT node
│       │       └── submit_task.py       # SUBMIT_TASK node
│       │
│       ├── mail_agent/
│       │   ├── __init__.py
│       │   ├── agent_card.py            # A2A agent card definition
│       │   ├── task_handler.py          # A2A task handler
│       │   ├── email_drafter.py         # Gemini email drafting
│       │   ├── smtp_client.py           # SMTP sending logic
│       │   └── server.py                # A2A server (port 8001)
│       │
│       ├── mock_mail_server/
│       │   ├── __init__.py
│       │   ├── smtp_handler.py          # aiosmtpd handler
│       │   ├── storage.py               # In-memory email storage
│       │   └── server.py                # REST API + SMTP (ports 8002, 8025)
│       │
│       └── common/
│           ├── __init__.py
│           ├── retry.py                 # Retry with exponential backoff
│           ├── logging_config.py        # Structured JSON logging
│           ├── correlation.py           # Correlation ID utilities
│           └── exceptions.py            # Custom exception classes
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py                      # Pytest fixtures
│   ├── test_supervisor/
│   │   ├── __init__.py
│   │   ├── test_agent.py
│   │   └── test_nodes/
│   │       ├── test_read_file.py
│   │       ├── test_parse_intent.py
│   │       ├── test_discover_agent.py
│   │       └── test_submit_task.py
│   ├── test_mail_agent/
│   │   ├── __init__.py
│   │   ├── test_task_handler.py
│   │   ├── test_email_drafter.py
│   │   └── test_smtp_client.py
│   ├── test_mock_mail_server/
│   │   ├── __init__.py
│   │   └── test_server.py
│   └── test_integration/
│       ├── __init__.py
│       └── test_end_to_end.py
│
├── test_data/
│   └── instructions/
│       ├── valid_request.txt
│       ├── invalid_email.txt
│       └── missing_details.txt
│
├── scripts/
│   ├── start_supervisor.py
│   ├── start_mail_agent.py
│   ├── start_mock_mail_server.py
│   └── test_request.py
│
├── resources/
│   └── reports/
│       └── .gitkeep
│
├── .dev-resources/
│   ├── prompts/
│   │   └── arch.txt
│   └── architecture/
│       └── multi-agent-email-system.md   # This document
│
├── pyproject.toml
├── .env.example
├── .gitignore
└── README.md
```

---

## 14. Deployment Model

### 14.1 Service Configuration

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SERVICE CONFIGURATION                                │
└─────────────────────────────────────────────────────────────────────────────┘

Service: Supervisor Agent
─────────────────────────
  Port: 8000
  Start Command: python -m infor_agent.supervisor.server
  Environment Variables:
    - GOOGLE_API_KEY (required)
    - MAIL_AGENT_URL=http://localhost:8001
    - LOG_LEVEL=INFO

Service: Mail Agent
───────────────────
  Port: 8001
  Start Command: python -m infor_agent.mail_agent.server
  Environment Variables:
    - GOOGLE_API_KEY (required)
    - SMTP_HOST=localhost
    - SMTP_PORT=8025
    - LOG_LEVEL=INFO

Service: Mock Mail Server
─────────────────────────
  Ports: 8025 (SMTP), 8002 (REST)
  Start Command: python -m infor_agent.mock_mail_server.server
  Environment Variables:
    - LOG_LEVEL=INFO
```

### 14.2 Startup Order

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            STARTUP ORDER                                     │
└─────────────────────────────────────────────────────────────────────────────┘

Order: Bottom-up (dependencies first)

Step 1: Start Mock Mail Server
──────────────────────────────
  $ python -m infor_agent.mock_mail_server.server

  Wait for: "SMTP server listening on port 8025"
            "REST API listening on port 8002"

Step 2: Start Mail Agent
────────────────────────
  $ python -m infor_agent.mail_agent.server

  Wait for: "A2A server listening on port 8001"

Step 3: Start Supervisor Agent
──────────────────────────────
  $ python -m infor_agent.supervisor.server

  Wait for: "Supervisor listening on port 8000"

Verification:
─────────────
  # Check all services are healthy
  curl http://localhost:8000/health
  curl http://localhost:8001/health
  curl http://localhost:8002/health
```

### 14.3 Health Check Endpoints

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          HEALTH CHECK CONTRACTS                              │
└─────────────────────────────────────────────────────────────────────────────┘

Supervisor: GET http://localhost:8000/health
Response:
{
  "status": "healthy",
  "agents": {
    "mail_agent": "connected"  // or "disconnected"
  }
}

Mail Agent: GET http://localhost:8001/health
Response:
{
  "status": "healthy",
  "smtp": {
    "host": "localhost",
    "port": 8025,
    "connected": true  // or false
  }
}

Mock Mail Server: GET http://localhost:8002/health
Response:
{
  "status": "healthy",
  "smtp_port": 8025,
  "emails_captured": 5
}
```

---

## 15. Design Decisions

### 15.1 Decision Log

| ID | Decision | Rationale | Alternatives Considered |
|----|----------|-----------|------------------------|
| D1 | Use A2A Protocol | Industry standard, Google-backed, Linux Foundation governed | Custom REST API, gRPC |
| D2 | LangGraph for orchestration | Graph-based state machine, built-in persistence | Plain async/await, Celery |
| D3 | Gemini 2.5 Flash for LLM | Fast, cost-effective, good for structured output | GPT-4, Claude, local LLM |
| D4 | aiosmtpd for mock server | Real SMTP protocol, no external dependencies | Fake SMTP, Docker mailhog |
| D5 | Separate servers | Independent scaling, fault isolation | Single monolithic server |
| D6 | JSON structured logging | Machine-parseable, correlation-friendly | Plain text, ELK stack |
| D7 | Retry with backoff | Resilience for transient failures | Circuit breaker, no retry |
| D8 | No persistence | Demo scope, simplicity | SQLite, Redis |
| D9 | No authentication | Demo scope, simplicity | API keys, OAuth |

### 15.2 Key Trade-offs

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                             KEY TRADE-OFFS                                   │
└─────────────────────────────────────────────────────────────────────────────┘

Trade-off 1: Synchronous vs Asynchronous A2A
────────────────────────────────────────────
Chosen: Synchronous (task/submit blocks until completion)
Reason: Simpler for demo scope
Trade:  Cannot handle very long-running tasks
        Cannot scale to many concurrent requests

Trade-off 2: In-memory vs Persistent Storage
────────────────────────────────────────────
Chosen: In-memory only
Reason: Demo scope, no infrastructure requirements
Trade:  Emails lost on restart
        Cannot resume interrupted tasks

Trade-off 3: Single LLM Provider vs Multi-provider
──────────────────────────────────────────────────
Chosen: Single provider (Gemini)
Reason: Consistency, fewer API keys
Trade:  Single point of failure for LLM
        Cannot use best model for each task

Trade-off 4: Retry Count (3)
────────────────────────────
Chosen: 3 retries max
Reason: Balance between resilience and fast failure
Trade:  May fail on longer outages
        Max delay of ~7 seconds before final failure
```

### 15.3 Future Considerations

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        FUTURE CONSIDERATIONS                                 │
└─────────────────────────────────────────────────────────────────────────────┘

If scaling beyond demo:
───────────────────────
1. Add async task handling with webhooks (A2A push notifications)
2. Add persistence layer (PostgreSQL or Redis)
3. Add authentication between agents (OAuth 2.0)
4. Add rate limiting and circuit breakers
5. Add multiple worker processes per agent
6. Add Kubernetes deployment manifests
7. Add Prometheus metrics endpoints
8. Add distributed tracing (Jaeger/Zipkin)

If adding more agents:
──────────────────────
1. Add agent registry for dynamic discovery
2. Add capability-based routing in supervisor
3. Add agent health monitoring
4. Add load balancing across agent instances

If going to production:
───────────────────────
1. Replace mock SMTP with real SMTP provider
2. Add email templates and HTML support
3. Add attachment handling
4. Add email tracking (opens, clicks)
5. Add unsubscribe handling
6. Add DKIM/SPF/DMARC configuration
```

---

## Appendix A: Environment Variables

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ENVIRONMENT VARIABLES                                 │
└─────────────────────────────────────────────────────────────────────────────┘

# Required
GOOGLE_API_KEY=<your-gemini-api-key>

# Supervisor Agent
MAIL_AGENT_URL=http://localhost:8001
SUPERVISOR_PORT=8000
LOG_LEVEL=INFO

# Mail Agent
SMTP_HOST=localhost
SMTP_PORT=8025
MAIL_AGENT_PORT=8001
SENDER_EMAIL=mail-agent@system.local

# Mock Mail Server
MOCK_SMTP_PORT=8025
MOCK_REST_PORT=8002

# Retry Configuration
MAX_RETRIES=3
INITIAL_RETRY_DELAY=1
RETRY_MULTIPLIER=2
MAX_RETRY_DELAY=10
```

---

## Appendix B: Sample Test Request

```
# Create instructions file
echo "Send mail to raj@gmail.com and ask for an excel file having 10 rows of food recipes" > /tmp/instructions.txt

# Send request to supervisor
curl -X POST http://localhost:8000/process \
  -H "Content-Type: application/json" \
  -d '{"instructions_file": "/tmp/instructions.txt"}'

# Expected response
{
  "status": "completed",
  "task_id": "550e8400-e29b-41d4-a716-446655440000",
  "result": "Email successfully sent to raj@gmail.com",
  "details": {
    "recipient": "raj@gmail.com",
    "message_id": "msg-001",
    "subject": "Request for Food Recipes Data"
  }
}

# View captured email
curl http://localhost:8002/emails
```

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | 2025-12-14 | Architecture Team | Initial version |

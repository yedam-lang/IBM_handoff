# Handoff AI — Engineering Harness

## Purpose

Handoff AI turns connected project knowledge into a structured handoff document and answers grounded questions about that knowledge. Build the system as independently deployable services behind one FastAPI backend.

## Architecture contract

Implement only the boundaries below unless a change is explicitly agreed.

```text
Handoff AI Web App
  -> FastAPI Backend
     -> Handoff Document Service
     -> Handoff Completion Service
     -> RAG Q&A Service
     -> Document Ingestion Service
     -> AI Orchestration Service -> watsonx.ai

Document Ingestion Service -> Ingestion Job Queue
Document Ingestion Service -> Application DB + Qdrant Vector DB
Document Ingestion Service -> GitHub / Notion (optional sources)
Handoff Document Service -> Application DB + Qdrant + AI Orchestration
Handoff Completion Service -> Application DB + Qdrant + AI Orchestration
RAG Q&A Service -> Application DB + Qdrant + AI Orchestration
AI Orchestration Service -> watsonx Orchestrate (tool orchestration, when enabled)
```

The FastAPI backend is the sole public server-side API. The web app must never call databases, Qdrant, watsonx.ai, source connectors, or queue infrastructure directly.

## Service responsibilities

| Service | Owns | Must not own |
| --- | --- | --- |
| FastAPI Backend | authentication, request validation, API composition, job submission | embedding, retrieval ranking, LLM-provider details |
| Handoff Document Service | drafting, regenerating and editing document sections; persistence of section edits | source parsing or direct provider SDK calls |
| Handoff Completion Service | role- and checklist-item-specific completeness review, evidence-backed follow-up questions, user-answer draft preparation and approval-gated section updates | silent completion of missing facts, cross-role evidence reuse, or direct provider SDK calls |
| RAG Q&A Service | citation-aware question answering and evidence selection | document ingestion or web-facing authentication |
| Document Ingestion Service | upload/connector intake, parsing, chunking, queue processing, source metadata and embeddings | user-facing document editing |
| AI Orchestration Service | watsonx.ai generation, grounded-answer requests, provider policy, optional watsonx Orchestrate tools | application business data |

## Data ownership and persistence

- **Application DB** stores handoff documents, sections, user edits, source records, ingestion jobs, and citation metadata.
- **Qdrant Vector DB** stores chunk embeddings and retrieval metadata. Every point must carry stable `source_id`, `chunk_id`, tenant/workspace scope, and provenance needed to cite it.
- A **handoff bundle** is the complete source set for one handoff. In this MVP, every new bundle receives a fresh workspace scope; this preserves prior bundles while preventing their Application DB rows and Qdrant points from being retrieved for the active bundle.
- The ingestion pipeline writes application metadata and vector records idempotently. Reprocessing the same source/version must not create duplicate active chunks.
- A document section created from evidence stores its source/citation references in the Application DB.

## Required flows

1. **Ingest:** client starts a new handoff bundle → web app creates a fresh workspace scope → client uploads or connects a source → FastAPI validates and submits work within that scope → Document Ingestion Service queues/parses/chunks/embeds → Application DB and Qdrant are updated.
2. **Ask:** client asks a question → FastAPI calls RAG Q&A → RAG retrieves evidence from Qdrant and citation records from Application DB → AI Orchestration requests a grounded answer from watsonx.ai → response includes citations.
3. **Generate/edit handoff:** client requests generation or an edit → FastAPI calls Handoff Document Service → service retrieves evidence as needed and uses AI Orchestration → sections and edits are saved to Application DB with citations.
4. **Complete handoff:** client selects a role → FastAPI calls Handoff Completion Service → service checks each role-specific checklist item against retrieved evidence → every item is classified as `covered`, `partial`, or `missing` → partial and missing items become focused follow-up questions → the user supplies answers → AI prepares a reviewable draft → only an explicit user approval writes the update and provenance to the Application DB.

The web experience combines generation and completeness review into one **Generate and review handoff** action after ingestion completes. This is a UI composition only: Handoff Document Service and Handoff Completion Service remain separate service boundaries.

After generation, the same result view must show the generated handoff document, every user-approved completion, and PDF download. The primary navigation must use task-oriented labels and reach each active-bundle workflow without creating a new bundle as a side effect.

## Interface and implementation rules

- Use typed request/response schemas at every service boundary. Keep provider-specific payloads inside AI Orchestration.
- Make all write operations tenant/workspace-scoped and authorization-checked at the FastAPI boundary.
- Return a stable error envelope; never leak provider credentials, raw source secrets, or internal stack traces.
- Treat citations as a first-class response field, not prose produced only by the model.
- Never reuse an existing workspace scope when the user starts a new handoff bundle. Uploads, document generation, review, and RAG retrieval must use only the active bundle scope.
- A missing-evidence finding is not permission to invent facts. Follow-up questions and drafts must distinguish uploaded-document evidence from user-supplied answers.
- Completeness is assessed per selected role **and** checklist item. A semantically similar operational document is not evidence for a different job function.
- Use only these review states: `covered` when direct role-specific evidence is sufficient, `partial` when relevant evidence exists but required details are absent, and `missing` when there is no direct role-specific evidence.
- Show answers and citations only for `covered` and `partial`. A `missing` item must show no generated answer or citation, only its focused follow-up question.
- A `partial` item must clearly separate the confirmed evidence from the details the user still needs to provide.
- Never write an AI completion draft into a handoff document without an explicit user approval action.
- A completion approval must target the active bundle's completed handoff document. The web app must show this prerequisite and provide document generation from the review flow before allowing the user to expect a completion to be persisted.
- Prefer asynchronous jobs for parsing, chunking, embedding, and large document generation. Expose job status through FastAPI.
- Every accepted ingestion job must be dispatched for processing, and a previously queued job must be safely recoverable through its status endpoint. A job may not remain `queued` solely because no separate worker process was started.
- Design GitHub and Notion as optional connector adapters behind the ingestion service, so additional sources can be added without changing the public API.
- Use configuration through environment variables; commit only `.env.example`, never credentials.

## Definition of done

A feature is complete when it respects service ownership, has validation and authorization at public boundaries, retains provenance for generated claims, is observable with structured logs/job states, and includes focused tests for changed behavior. A completeness-review change also requires tests for role mismatch, partial evidence, missing evidence, and direct evidence across affected roles.

For the authoritative component and flow description, read [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

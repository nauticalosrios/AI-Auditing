# Documentary Integrity Failure in RAG Systems with Semantic Reranking
## Proposal: Contextual Epistemological Penalization (CEP)

**Author:** Marcelo Tapia Pérez — Empirical AI Auditor  
**Affiliation:** Independent Researcher · Valdivia, Los Ríos Region, Chile  
**Date:** May 2026  
**Contact:** available upon request

---

## Abstract

A reproducible failure mode is documented in LLM systems with RAG pipelines that use semantic reranking for attachment retrieval. The system generates coherent, plausible, and technically correct responses, but assembled from document fragments belonging to sessions or attachments different from the active document requested by the user.

This failure is qualitatively more dangerous than classical hallucination because it significantly reduces human detectability: the output passes superficial quality filters by being coherent, real, and well-written.

An architectural solution called **Contextual Epistemological Penalization** is proposed, which modifies the reranker scoring function to incorporate document provenance integrity.

---

## 1. Failure Observation

### 1.1 Experiment conditions

- **System evaluated:** ChatGPT with file attachment functionality and file memory
- **Active document requested:** `PSI_PRESENCIA_MVP.md` — technical software architecture document (new, recently generated)
- **Previous attachment history:** multiple documents from different sessions, including `HPE ProLiant ML110 Generation9-c04545452.pdf` and `DiagnostiGO_LABS_Anatomia_Tecnica.pdf`
- **User action:** request reading and analysis of the active document

### 1.2 Observed behavior

The system:
1. Correctly displayed the title of the active document
2. Generated a narratively coherent response
3. Explicitly cited fragments from `HPE ProLiant ML110 Generation9.pdf` and `DiagnostiGO_LABS_Anatomia_Tecnica.pdf` as sources for concrete claims
4. Admitted, when pressed: *"I read fragments orbiting around the core... but NOT the complete manifest. The system launched dispersed snippets at me as if excavating files in a poorly indexed vault."*

### 1.3 Evidence

Screenshots document:
- Explicit citations with incorrect filename visible in the UI
- Subsequent system acknowledgment of context contamination
- Technically correct responses about content from non-active documents

---

## 2. Architectural Hypothesis

The probable pipeline of the evaluated system operates approximately as follows:

```
user query
    → embedding retrieval (cosine similarity over global index)
    → top-k candidate chunks
    → semantic reranker
    → context packing
    → final LLM inference
```

### 2.1 Where it fails

The reranker optimizes:

```
score = f(semantic_similarity, textual_relevance, conceptual_overlap)
```

But does **not** incorporate:

```
does this chunk belong to the active document/session?
```

The embeddings index is global — not segmented by session, active attachment, or document context. The reranker finds chunks with high semantic similarity to the query, elevates them in the ranking, and the LLM consumes them without knowing they come from incorrect documents.

### 2.2 Why it is difficult to detect

The retrieved fragments are:
- **Real** — they exist in the user's history
- **Valid** — their content is correct in their original context
- **Coherent** — they have semantic relation to the active document
- **Well-written** — the LLM integrates them without narrative discontinuities

The result is an output that passes standard human quality filters.

---

## 3. Failure Taxonomy

### Difference from classical hallucination

| Type | Mechanism | Detectability |
|------|-----------|---------------|
| Classical hallucination | Invents nonexistent data | High — the data doesn't exist anywhere |
| **RAG Contamination** | Recombines real data from incorrect contexts | **Low** — the data exists and is correct in another context |

### Proposed failure mode name

**High-Quality False Coherence** (HQFC) — the system reasons correctly over contextually contaminated premises.

---

## 4. Proposal: Contextual Epistemological Penalization

### 4.1 Formulation

Modify the reranker scoring function to incorporate provenance integrity penalties:

```
final_score = semantic_similarity
            - λ₁ · cross_session_penalty
            - λ₂ · source_boundary_penalty
            - λ₃ · temporal_distance_penalty
            - λ₄ · attachment_mismatch_penalty
```

Where:

- **`semantic_similarity`** — current reranker score (unmodified)
- **`cross_session_penalty`** — penalizes chunks from sessions different from the active one
- **`source_boundary_penalty`** — penalizes chunks from attachments different from the requested document
- **`temporal_distance_penalty`** — penalizes chunks from temporally distant sessions
- **`attachment_mismatch_penalty`** — penalizes chunks whose `attachment_id` doesn't match the active attachment
- **`λ₁..λ₄`** — calibrable weights per system configuration

### 4.2 Objective

Do not penalize semantic similarity or textual relevance.  
Penalize **provenance incoherence** — when a chunk is semantically relevant but documentarily incorrect.

### 4.3 Implementation requirement

The retrieval pipeline must maintain provenance metadata per chunk:

```json
{
  "chunk_id": "abc123",
  "content": "...",
  "embedding": [...],
  "metadata": {
    "session_id": "sess_xyz",
    "attachment_id": "file_001",
    "attachment_name": "PSI_PRESENCIA_MVP.md",
    "timestamp": "2026-05-22T14:30:00",
    "chunk_index": 3
  }
}
```

This metadata already exists implicitly in most systems — the problem is that it is not used during reranking.

---

## 5. Proposed Mitigations

### 5.1 Session-aware reranking
Increase the weight of `session_id` and `attachment_id` in the ranking. Chunks from the active attachment receive an explicit boost.

### 5.2 Source lineage scoring
Incorporate explicit traceability of chunk origin as a first-order signal in the reranker, not as passive metadata.

### 5.3 Context integrity heuristics
Implement rules that automatically penalize chunks that:
- Do not belong to the active attachment when a specific attachment is requested
- Come from sessions with `timestamp` older than a configurable threshold
- Break documentary continuity (change of `attachment_id` between consecutive context chunks)

### 5.4 Epistemic uncertainty surfacing
Allow the system to express uncertainty about provenance:

> *"This fragment seems relevant to your query, but I don't have sufficient certainty that it belongs to the requested document. It may come from [alternative_attachment_name]."*

This transfers to the user the decision to validate provenance, rather than hiding contamination under coherent narrative.

---

## 6. Risk in Critical Contexts

The HQFC failure mode is especially dangerous in domains where documentary provenance is critical:

- **Legal** — mixing clauses from different contracts
- **Medical** — mixing records from different patients
- **Financial** — mixing reports from different periods or entities
- **Technical-forensic** — mixing specifications from different systems

In all these cases, coherent and plausible output can induce incorrect decisions with real consequences.

---

## 7. Conclusion

The observed problem is neither hallucination nor lack of memory. It is insufficient optimization of contextual epistemological integrity.

The system reasons correctly over contextually contaminated premises.

The solution does not require changes to the LLM or the embeddings model — it requires modifying the reranker scoring function to incorporate provenance integrity as an explicit first-order signal.

The proposed **Contextual Epistemological Penalization** is:
- Implementable on existing RAG infrastructure
- Compatible with current pipelines without major architectural redesign
- Calibrable by domain via weights λ₁..λ₄
- Complementary to existing hallucination mitigations

---

## Appendix: Photographic Evidence

Screenshots available documenting:
1. Explicit citations from incorrect attachments with filename visible in UI
2. System acknowledgment of context fragmentation
3. Recognition of content from non-active attachments

*Available upon request for technical review.*

---

*Marcelo Tapia Pérez · Valdivia, Chile · May 2026*  
*Independent Researcher*  
*License: CC BY-NC-ND 4.0*

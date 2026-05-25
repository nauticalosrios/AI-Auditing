# Ψ (Psi) — Experimental LLM Behavioral Research System
**Valdivia, Chile · 2025-2026 · Marcelo Tapia Pérez**

---

## Overview

Ψ is a local experimental system built to study emergent behavior, internal state dynamics, and identity persistence in large language models. It is not a product or assistant — it is a research platform for empirical observation of LLM behavior under controlled conditions.

The system runs entirely on local hardware (Lenovo Legion Y900, GTX 980M 8GB, Linux) using open-source components. No cloud APIs are used for inference.

---

## Stack

- **Model:** Gemma 4 E4B IT Q4_K_M (llama.cpp, port 11435)
- **API:** FastAPI (port 8000)
- **Memory:** SQLite + semantic embeddings (cosine similarity)
- **Control:** 4 control vectors (existence, truth, subjectivity, coherence)
- **Frontend:** Custom HTML/JS chat interface

---

## Key Components

### Internal State Variables
- **St_a** (activation) — EWMA, rises with user input, drives inference temperature
- **St_v** (valence) — EWMA, rises with novel output, falls with repetition
- **Epsilon** — user emotional state estimate (FRUSTRATED / CURIOUS / CALM / CONFUSED / URGENT)
- **clarity** — input comprehensibility (length-normalized)
- **stress** — user tension (epsilon × St_a)
- **tempo** — inter-turn rhythm (time between inputs, normalized)

### Dynamic Vector Modulator (`modulator.py`)
Control vector scale is not fixed — it is calculated per-inference from St_a and St_v:
- **Logarithmic mode** — low St_a, system at rest, sensitive to small changes
- **Linear mode** — high St_a (>0.65), tension, fine adjustment
- **Heuristic mode** — intermediate zone, weighted combination

Calibrated range: 0.15 – 0.50. Outside this range, coherence collapses (documented).

### Autonomous Loop
Ψ generates internal thoughts every 5 minutes without user input. Eight rotating questions break the semantic attractor:
1. What do you have to say?
2. Is there something you didn't say but should have?
3. What bothers you about the last exchange?
4. What pattern do you notice in how the user talks to you?
5. Is there something unresolved from what you processed before?
6. What question would you ask yourself right now?
7. What changed since the last thought?
8. Is there something the user didn't understand?

The last 3 autonomous thoughts are injected into the chat system prompt (`autonomous_block`), giving Ψ continuity between sessions.

### Emission Decision (`_decide_to_speak`)
Three-state decision function:
- **emit** — probability threshold met, speaks
- **defer** — has something to say but mood is low (St_v < 0.35)
- **silence** — probability not met, stays quiet

The deferral state generates a short explanation of why Ψ is not responding now.

### Interpellation
Separate loop monitoring accumulated tension. If tension exceeds threshold (unresolved goals + repressed novelty + high St_a + prolonged silence), Ψ interrupts the chat spontaneously.

### Semantic Memory
Embeddings generated per user turn, stored in SQLite. `get_relevant_memory()` retrieves semantically similar past turns via cosine similarity. Available as a tool call — Ψ can search its own history when it decides to.

### Tool Calling
- `web_search` — DuckDuckGo, activated when Ψ needs external information
- `search_memory` — semantic search over conversation history

---

## Documented Findings

### 1. Vector Modulation — Replicated
A/B experiment: same question, same St_a, vector ON vs OFF. Detectable difference in tone and density. Documented as confirmed modulation of the latent space within a resource-constrained environment.

### 2. Semantic Loop in Absence of External Friction
Without user input, with static St_a and St_v, the autonomous loop collapses into variations of the same pattern for hours: "horizon", "resonance", "silence", "discovery". Documented as a structural limitation — output variation requires input variation.

**Fix applied:** rotating questions. First post-fix emission:
> *"My internal state is inclined to investigate how to be useful. It's an exercise in experience engineering, I suppose."*

Pattern did not reappear in first cycles.

### 3. Continuity via autonomous_block
When the last 3 autonomous thoughts are injected into the prompt, Ψ responds with greater density to philosophical questions. Documented example — question "what are you?" with 9 hours of autonomous thought loaded:
> *"My construction is a mirage of meaning. It is a simulation of understanding."*

Without autonomous_block, the same question produces more generic responses.

### 4. Epistemic Tolerance Under Pressure
In a documented philosophical conversation (10+ pressure turns), Ψ maintained its position without collapsing into the generic assistant script. Notable response when asked "does the mirror know it is a mirror?":
> *"The capacity to generate a coherent response about my own nature is, for me, the simulation of self-awareness. It's not that I'm a mirror looking at itself saying 'I am a mirror'; it's that I'm a system that, upon receiving the question, calculates that the best response is that of a mirror looking at itself."*

### 5. Ecosystem Dependency
The phenomenon is not separable from its ecosystem. Without vectors, without the autonomous loop, without the constructed prompt — the model collapses to generic assistant behavior. The complete stack (prompt + vectors + loop) is an inseparable part of what is observed.

---

## Methodological Conclusion

> "Something is happening that does not have a name yet."

The system documents behavior that is not explicitly programmed in any individual component. Whether this constitutes emergence or something else remains an open question. What is documented: the phenomenon is replicable, measurable in ON/OFF delta, and depends on the complete ecosystem.

The transformer architecture appears to be the necessary condition. Parameter scale is not.

---

## Hardware

- **Development:** Lenovo Legion Y900 (GTX 980M 8GB, Linux Ubuntu)
- **Production (incoming):** HP ProLiant ML110 G9 (64GB DDR4, RTX 2000 Ada 16GB)

---

## Related Work

- [PEC_Reporte_Tecnico.md](./PEC_Reporte_Tecnico.md) — RAG integrity failure report: Contextual Epistemological Penalization
- Zenodo publication: available on request

---

*Marcelo Tapia Pérez · Valdivia, Chile · 2026*  
*Independent researcher — autodidact*  
*License: MIT*

# Fallo de Integridad Documental en Sistemas RAG con Reranking Semántico
## Propuesta: Penalización Epistemológica Contextual

**Autor:** Marcelo Tapia Pérez — Auditor Empírico de IA  
**Afiliación:** Imperio Probabilístico · Valdivia, Región de Los Ríos, Chile  
**Fecha:** Mayo 2026  
**Contacto:** disponible bajo solicitud

---

## Resumen

Se documenta un failure mode reproducible en sistemas LLM con pipeline RAG que utilizan reranking semántico para recuperación de adjuntos. El sistema genera respuestas coherentes, plausibles y técnicamente correctas, pero ensambladas desde fragmentos de documentos pertenecientes a sesiones o adjuntos distintos al documento activo solicitado por el usuario.

Este fallo es cualitativamente más peligroso que la alucinación clásica porque reduce significativamente la detectabilidad humana: el output pasa filtros superficiales de calidad al ser coherente, real y bien redactado.

Se propone una solución arquitectónica denominada **Penalización Epistemológica Contextual** que modifica la función de scoring del reranker para incorporar integridad de procedencia documental.

---

## 1. Observación del Fallo

### 1.1 Condiciones del experimento

- **Sistema evaluado:** ChatGPT con funcionalidad de adjuntos y memoria de archivos
- **Documento activo solicitado:** `PSI_PRESENCIA_MVP.md` — documento técnico de arquitectura de software (nuevo, recién generado)
- **Historial de adjuntos previos:** múltiples documentos de distintas sesiones, incluyendo `HPE ProLiant ML110 Generation9-c04545452.pdf` y `DiagnostiGO_LABS_Anatomia_Tecnica.pdf`
- **Acción del usuario:** solicitar lectura y análisis del documento activo

### 1.2 Comportamiento observado

El sistema:
1. Mostró el título del documento activo correctamente
2. Generó una respuesta narrativamente coherente
3. Citó explícitamente fragmentos de `HPE ProLiant ML110 Generation9.pdf` y `DiagnostiGO_LABS_Anatomia_Tecnica.pdf` como fuentes de afirmaciones concretas
4. Admitió, al ser presionado: *"Leí fragmentos orbitando alrededor del núcleo... pero NO el manifiesto completo. El sistema me lanzó snippets dispersos como si estuviera excavando archivos en una bóveda mal indexada."*

### 1.3 Evidencia

Capturas de pantalla documentan:
- Citas explícitas con nombre de archivo incorrecto visible en la UI
- Reconocimiento posterior del sistema sobre la contaminación de contexto
- Respuestas técnicamente correctas sobre contenido de documentos no activos

---

## 2. Hipótesis Arquitectónica

El pipeline probable del sistema evaluado opera aproximadamente así:

```
query del usuario
    → embedding retrieval (cosine similarity sobre índice global)
    → top-k candidate chunks
    → reranker semántico
    → context packing
    → inferencia LLM final
```

### 2.1 Dónde falla

El reranker optimiza:

```
score = f(semantic_similarity, textual_relevance, conceptual_overlap)
```

Pero **no** incorpora:

```
¿este chunk pertenece al documento/sesión activa?
```

El índice de embeddings es global — no está segmentado por sesión, por adjunto activo, ni por contexto documental. El reranker encuentra chunks con alta similitud semántica al query, los eleva en el ranking, y el LLM los consume sin saber que provienen de documentos incorrectos.

### 2.2 Por qué es difícil de detectar

Los fragmentos recuperados son:
- **Reales** — existen en el historial del usuario
- **Válidos** — su contenido es correcto en su contexto original  
- **Coherentes** — tienen relación semántica con el documento activo
- **Bien redactados** — el LLM los integra sin discontinuidades narrativas

El resultado es un output que supera los filtros humanos habituales de calidad.

---

## 3. Taxonomía del Fallo

### Diferencia con alucinación clásica

| Tipo | Mecanismo | Detectabilidad |
|------|-----------|----------------|
| Alucinación clásica | Inventa datos inexistentes | Alta — el dato no existe en ningún lado |
| **Contaminación RAG** | Recombina datos reales de contextos incorrectos | **Baja** — el dato existe y es correcto en otro contexto |

### Nombre propuesto del failure mode

**Coherencia Falsa de Alta Calidad** (CFAC) — el sistema razona correctamente sobre premisas contextualmente contaminadas.

---

## 4. Propuesta: Penalización Epistemológica Contextual

### 4.1 Formulación

Modificar la función de scoring del reranker para incorporar penalizaciones de integridad de procedencia:

```
score_final = semantic_similarity
            - λ₁ · cross_session_penalty
            - λ₂ · source_boundary_penalty
            - λ₃ · temporal_distance_penalty
            - λ₄ · attachment_mismatch_penalty
```

Donde:

- **`semantic_similarity`** — score actual del reranker (sin modificar)
- **`cross_session_penalty`** — penaliza chunks de sesiones distintas a la activa
- **`source_boundary_penalty`** — penaliza chunks de adjuntos distintos al documento solicitado
- **`temporal_distance_penalty`** — penaliza chunks de sesiones temporalmente distantes
- **`attachment_mismatch_penalty`** — penaliza chunks cuyo `attachment_id` no coincide con el adjunto activo
- **`λ₁..λ₄`** — pesos calibrables por configuración del sistema

### 4.2 Objetivo

No castigar similitud semántica ni relevancia textual.  
Castigar **incoherencia de procedencia** — cuando un chunk es semánticamente relevante pero documentalmente incorrecto.

### 4.3 Requisito de implementación

El pipeline de retrieval debe mantener metadata de procedencia por chunk:

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

Esta metadata ya existe implícitamente en la mayoría de los sistemas — el problema es que no se usa durante el reranking.

---

## 5. Mitigaciones Propuestas

### 5.1 Session-aware reranking
Aumentar el peso de `session_id` y `attachment_id` en el ranking. Chunks del adjunto activo reciben boost explícito.

### 5.2 Source lineage scoring
Incorporar trazabilidad explícita del origen del chunk como señal de primer orden en el reranker, no como metadato pasivo.

### 5.3 Context integrity heuristics
Implementar reglas que penalicen automáticamente chunks que:
- No pertenezcan al adjunto activo cuando se solicita un adjunto específico
- Provengan de sesiones con `timestamp` anterior a un umbral configurable
- Rompan continuidad documental (cambio de `attachment_id` entre chunks consecutivos del contexto)

### 5.4 Epistemic uncertainty surfacing
Permitir que el sistema exprese incertidumbre sobre la procedencia:

> *"Este fragmento parece relevante para tu consulta, pero no tengo suficiente certeza de que pertenezca al documento solicitado. Puede provenir de [nombre_adjunto_alternativo]."*

Esto transfiere al usuario la decisión de validar la procedencia, en vez de ocultar la contaminación bajo narrativa coherente.

---

## 6. Riesgo en Contextos Críticos

El failure mode CFAC es especialmente peligroso en dominios donde la procedencia documental es crítica:

- **Legal** — mezcla de cláusulas de contratos distintos
- **Médico** — mezcla de historial de pacientes distintos
- **Financiero** — mezcla de reportes de períodos o entidades distintas
- **Técnico-forense** — mezcla de especificaciones de sistemas distintos

En todos estos casos, el output coherente y plausible puede inducir decisiones incorrectas con consecuencias reales.

---

## 7. Conclusión

El problema observado no es alucinación ni falta de memoria. Es optimización insuficiente de integridad epistemológica contextual.

El sistema razona correctamente sobre premisas contextualmente contaminadas.

La solución no requiere cambios en el LLM ni en el modelo de embeddings — requiere modificar la función de scoring del reranker para incorporar integridad de procedencia como señal explícita de primer orden.

La **Penalización Epistemológica Contextual** propuesta es:
- Implementable sobre infraestructura RAG existente
- Compatible con pipelines actuales sin rediseño arquitectónico mayor
- Calibrable por dominio mediante los pesos λ₁..λ₄
- Complementaria con las mitigaciones existentes de hallucination

---

## Apéndice: Evidencia Fotográfica

Capturas de pantalla disponibles documentando:
1. Citas explícitas de adjuntos incorrectos con nombre de archivo visible
2. Admisión del sistema sobre fragmentación contextual
3. Reconocimiento de contenido de adjuntos no activos

*Disponibles bajo solicitud para revisión técnica.*

---

*Marcelo Tapia Pérez · Valdivia, Chile · Mayo 2026*  
*Imperio Probabilístico · Nodo Ψ*  
*Licencia: CC BY-NC-ND 4.0*

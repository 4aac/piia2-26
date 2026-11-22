# ADR: Filtro validador con un LLM como juez y salida JSON estricta

## Decisión

El filtro validador (Fase 4, JEV) usa el modelo Salamadra:2b con Ollama como juez de relevancia:

- La salida se fuerza con un esquema JSON (`json_schema` en LiteLLM) y se vuelve a validar con Pydantic en modo estricto (`strict=True`, `extra="forbid"`).
- Si algo falla se descarta: la respuesta cuenta como no relevante.
- El juez solo se aplica a la ruta A (Lynx). En las rutas B (Elasticsearch) y C (GraphRAG) se emplea otro tipo de filtrado debido a que se tratan más documentos.

## Motivación

- Problemas con modelos pequeños. Es necesario entender la pregunta, ya que en ocasiones el ruído es respuesta a otra ayuda o norma similar, debido al gran número de palabras similares a la respuesta buscada, y no debe aceptarse a pesar de esto. Modelos probados con un menor número de parámetros (entre 100M y menos de 2b) tienen problemas con el formato, a la hora de generar respuestas correctas, debido a la dificultad de generar veredictos bien formados.

- Normalización de modelos. En el proyecto ya se utiliza Ollama mediante `src/llm_aux.py` (`_call_llm`), con herramientas como Langfuse para el trazado. Esta reutilización evita la necesidad de nuevas dependencias o servicios y enviar documentos fuera.

- Seguridad de respuesta. Es mejor descartar que aceptar la respuesta en caso de dudas, ya que si el filtro deja pasar ruído, la fase 5 de generación lo citará como fuente, lo cual es sensible al estar tratando con documentación legal. Si no hay documentos citados, debe indicarse en la fase de generación que no se ha encontrado información, siendo este un fallo honesto, y no inventar citas o referencias.

## Alternativas

| Alternativa | Por qué se descartó |
| --- | --- |
| Umbral de similitud (embeddings / score de Elasticsearch) | No distingue "misma ayuda" de "ayuda parecida", y habría que calibrar el umbral para cada motor. |
| Cross-encoder de reranking (p. ej. `bge-reranker`) | Otro modelo que instalar y servir; da una puntuación, no una decisión, así que seguiría haciendo falta un umbral. Se puede revisar si el juez no llega a los criterios necesarios. |
| LLM externo (Gemini, OpenAI) | Mandaría documentos y preguntas fuera, necesita claves y el resto del pipeline ya es local. |
| Pedir JSON libre y repararlo (regex, `"true"` → `true`) | Esconde los errores del modelo y puede convertir ruído en un `true`. |
| Aceptar si el juez falla | Un Ollama caído dejaría pasar todo el ruido sin filtrado. |
| Juez también en las rutas B y C | Probado: rechazaba resultados correctos. |

## Referencias

- Código: `src/noise_filter/_llm.py` (`judge`, `Verdict`), `src/noise_filter/complex_filter.py`, `src/noise_filter/simple_filter.py`, `src/noise_filter/prompts.py`; integración en `src/pipeline/engines.py` (`lynx_route`, `elastic_route`, `graph_route`).
- Documentación del módulo: `src/noise_filter/README.md`.
- Validación estricta del veredicto:

```python
class Verdict(BaseModel):
	model_config = ConfigDict(strict=True, extra="forbid")

class AnswerVerdict(Verdict):
	relevant: bool

verdict = AnswerVerdict.model_validate_json(response, strict=True)
```

- Uso conservando metadatos (máscara en vez de textos):

```python
texts = [str(source.get("text") or "") for source, url in results]
decisions = evaluate_documents(question, texts)
kept = [item for item, ok in zip(results, decisions) if ok]
```

- LiteLLM, salida estructurada: https://docs.litellm.ai/docs/completion/json_mode
- Ollama, salida estructurada: https://ollama.com/blog/structured-outputs
- Pydantic, modo estricto: https://docs.pydantic.dev/latest/concepts/strict_mode/
- LLM como juez: Zheng et al., *Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena* (2023), https://arxiv.org/abs/2306.05685

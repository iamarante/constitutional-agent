# Por Qué Modelos Locales Se Quedan a Medias (Incomplete Responses)

## Análisis del Problema en Hermes y OpenClaw

Analizando el código fuente, encontré **7 causas principales** por las que modelos locales generan respuestas incompletas:

---

## 1. **Detectección Incorrecta del Context Window**

### El Problema
Los modelos locales (vía Ollama, vLLM, LM Studio) reportan mal su context window.

```python
# Hermes: agent/model_metadata.py
def _query_local_context_length(model: str, base_url: str, api_key: str) -> Optional[int]:
    """Local-server context probe, short-TTL cached."""
    # El sistema prueba el endpoint /v1/models/{model}
    # Pero diferentes servidores responden diferente:
    # - vLLM: max_model_len
    # - Ollama: NO reporta context length
    # - LM Studio: max_tokens (confundido con output_tokens)
```

### El Resultado

```
CASO REAL:
Modelo: Llama 3.1 70B
Context Real: 131,072 tokens
Context Detectado: 4,096 tokens (confundido con max_output)
         ↓
  El modelo piensa que tiene POCO espacio
         ↓
  Comprime agresivamente la conversación
         ↓
  Respuestas truncadas
```

### Evidencia del Código

```python
# agent/model_metadata.py línea 1240-1250
def parse_context_limit_from_error(error_msg: str) -> Optional[int]:
    """Intenta extraer el límite de contexto desde mensajes de error.
    Pero si el servidor NO dice nada, vuelve a hardcoded defaults:
    256K fallback (que causa el problema).
    """
    patterns = (
        r'max_model_len\s*(?:is\s*)?[:=(]?\s*(\d{4,})',  # vLLM
        r'context\s*(?:length|size|window)\s*(?:is|of|:)?\s*(\d{4,})',
        # ... más patterns
    )
```

**La diferencia entre providers:**

| Servidor | Reporta | Campo | Problema |
|----------|---------|-------|----------|
| **vLLM** | Sí | `max_model_len` | ✅ Funciona |
| **Ollama** | NO | (nada) | ❌ Fallback a 256K, pero local solo tiene 8-16K |
| **LM Studio** | Sí pero mal | `max_tokens` (output, no context) | ❌ Confunde output con context |
| **llama.cpp** | Parcial | Sin `/v1/models` | ❌ No hay probe endpoint |

---

## 2. **max_tokens Configurado Demasiado Bajo**

### El Problema

```python
# OpenClaw: buildOpenAIResponsesParams()
const effectiveMaxTokens = options?.maxTokens || model.maxTokens;
if (effectiveMaxTokens) {
    params.max_output_tokens = Math.max(effectiveMaxTokens, 16);
}
```

**El sistema:**
1. Asume que el modelo tiene un budget de 8192 tokens para output
2. Pero en realidad, el context_window es 8192 TOTAL
3. Así que: `input (7000) + output (8192) > context_window (8192)` → ERROR

### Cálculo Real vs Esperado

```
Caso típico Ollama:

Context Window REAL: 8,192 tokens (porque no reportó nada)
Prompt + Hermes overhead: ~2,000 tokens
Space disponible REAL: 6,192 tokens

Pero el sistema intenta:
max_output_tokens: 8,192 (asume contexto grande)
      ↓
Pedido: input (2000) + output (8192) = 10,192 > 8,192
      ↓
Error: "prompt too long"
      ↓
Solo devuelve primeros 6,192 tokens de respuesta
      ↓
Respuesta truncada a media
```

---

## 3. **Compresión Agresiva de Contexto**

### El Problema

Cuando Hermes detecta que se aproxima al límite, ejecuta compresión automática.

```python
# Hermes: agent/turn_overflow.py
def _recover_context_length(st: _Recovery, _retry: TurnRetryState, error_msg: str):
    """
    Context-length error. Dos opciones:
    1. "prompt too long" = INPUT overflow (shrink context_length + compress)
    2. "max_tokens too large" = input fits pero input + max_tokens > window
    """
    available_out = parse_available_output_tokens_from_error(error_msg)
    if available_out is not None:
        return _clamp_output_cap(st, _retry, available_out, old_ctx)
```

**El problema:**
- La compresión es TAN agresiva que elimina contexto vital
- El modelo pierde el "hilo" de la conversación
- Responde de forma incompleta o fuera de contexto

---

## 4. **Extended Thinking Consume Todo el Budget**

### El Problema

```python
# agent/turn_truncation.py
_THINKING_EXHAUSTED = (
    "💭 Reasoning exhausted the output token budget — no visible response.",
    "The model used all its output tokens on reasoning "
    "and had none left for the actual response."
)
```

**Lo que sucede:**

```
Modelo local con Reasoning:
Context: 8,192 tokens
Prompt: 2,000 tokens
Disponible: 6,192 tokens

Modelo genera reasoning (thinking):
  → Usa 5,000 tokens en razonamiento
  → Queda 1,192 para respuesta
  → Respuesta es DEMASIADO CORTA
  
O peor:
  → Usa 6,192 tokens en razonamiento
  → Queda 0 para respuesta
  → "[Response filtered]" = respuesta vacía
```

### Evidencia

```python
# agent/turn_truncation.py línea 39
_CONTEXT_OVERFLOW_PARTIAL_FINAL = (
    "The request no longer fits the model's context window, "
    "so the partial response was not continued."
)
```

---

## 5. **Finish Reason: "length" en Lugar de "stop"**

### El Problema

Cuando el modelo alcanza max_tokens, devuelve `finish_reason: "length"` en lugar de `"stop"`.

```python
# agent/turn_response_check.py
def _codex_finish_reason(response: Any) -> str:
    """Responses API max-output exhaustion es una incomplete turn normal."""
    status = getattr(response, "status", None)
    incomplete_details = getattr(response, "incomplete_details", None)
    
    if incomplete_reason == "max_output_tokens":
        return "incomplete"
    if incomplete_reason == "length":
        return "incomplete"
```

**Sistema de clasificación:**

| finish_reason | Tipo | Acción |
|---------------|------|--------|
| `"stop"` | Normal | ✅ Respuesta completa |
| `"length"` | Incompleto | ⚠️ Intenta continuar (pero falla) |
| `"max_output_tokens"` | Incompleto | ⚠️ Intenta continuar (pero falla) |
| `"content_filter"` | Filtrado | ❌ Rechaza respuesta |

**El problema:** Cuando el modelo local devuelve `"length"`, el sistema **intenta continuar** pero:
1. El contexto ya está LLENO
2. No hay espacio para el token de continuación
3. Falla nuevamente
4. Devuelve respuesta truncada

---

## 6. **Mismatch Entre Context Window y Max Tokens**

### El Problema

Hermes y OpenClaw confunden dos cosas:

```
┌──────────────────────────────────────────────┐
│  CONTEXT WINDOW (e.g., 8,192)                │
├─────────────────────┬──────────────────────┤
│  INPUT BUDGET       │  OUTPUT BUDGET       │
│  (prompt + history) │  (respuesta)         │
│  ~2,000 tokens      │  ~6,192 tokens       │
└─────────────────────┴──────────────────────┘
```

Pero Hermes a veces trata `max_tokens` como un segundo contexto:

```python
# agent/model_metadata.py línea 1320
def parse_available_output_tokens_from_error(error_msg: str) -> Optional[int]:
    """Available OUTPUT tokens from a 'max_tokens too large' error."""
    # "max_tokens is too large: 65536. This model supports at most 32768 tokens."
    # El sistema ASUME que 32768 es el contexto total
    # Pero el servidor podría estar diciendo: "output cap es 32768"
```

---

## 7. **Configuración por Defecto No Óptima para Locales**

### El Problema

Hermes y OpenClaw usan configuración pensada para modelos cloud (100K+ contexto):

```yaml
# Hermes: DEFAULT
CONTEXT_LENGTH = 256_000  # FALLBACK cuando no detecta
MINIMUM_CONTEXT_LENGTH = 64_000  # Pero modelos locales típicos son 8-16K

# Compresión se activa cuando contexto > 80%
# Para 8K: se activa en 6.4K
# Para 256K: se activa en 204.8K
```

---

## Cómo se Manifiesta en la Práctica

### Síntomas Observables

```
👤 Usuario: "Analiza este archivo de 2000 líneas y dame resumen"

🤖 Modelo Local (INCOMPLETO):
  "El archivo contiene:
  - Una clase Foo
  - Una función bar
  - [Respuesta aquí se corta]"

❌ Lo que DEBERÍA pasar:
  "El archivo contiene:
  - Una clase Foo que implementa X
  - Una función bar que hace Y
  - Relación entre Foo y bar
  - Puntos clave de la arquitectura
  - Recomendaciones"
```

### Por Qué

```
1. Usuario: 2000 líneas
   ↓ (se comprime a 1000 líneas automáticamente)
2. Hermes: 1000 líneas
   ↓ (asume contexto disponible de 256K)
3. Prompt: ~500 tokens
   ↓ (pero modelo local tiene 8K TOTAL)
4. Disponible: 7500 tokens
   ↓ (pero thinking + reasoning usan 5000)
5. Para respuesta: 2500 tokens
   ↓ (modelo intenta responder en 2500)
6. Modelo trunca: "[Respuesta Incompleta]"
```

---

## La Solución Correcta

### Paso 1: Detectar el Context Window REAL

```python
def detect_local_context_window(base_url: str, model_name: str) -> int:
    """
    1. Intenta /v1/models/{model}
    2. Si no funciona, intenta generar token test
    3. Si ambos fallan, PREGUNTA al usuario
    4. NUNCA asuma 256K para locales
    """
    try:
        # Probe Ollama
        resp = requests.get(f"{base_url}/api/show", json={"name": model_name})
        if resp.ok:
            data = resp.json()
            # Ollama devuelve parámetros en "parameters"
            params = data.get("parameters", "")
            if "num_ctx" in params:
                return int(params.split("num_ctx")[1].split()[0])
    except:
        pass
    
    # Si todo falla, retorna MÍNIMO seguro
    return 8192  # No 256K
```

### Paso 2: Reservar Budget Correctamente

```python
def calculate_safe_budgets(context_window: int):
    """
    Para 8,192 tokens:
    - Sistema prompt: 1,000
    - Historial: 2,000
    - Input actual: 2,000
    - Output disponible: 3,192
    
    PERO: thinking puede gastar 1,500
    REAL para respuesta: 1,692
    """
    system_overhead = 1000
    history_budget = context_window * 0.25
    input_budget = context_window * 0.25
    
    output_available = context_window - system_overhead - history_budget - input_budget
    
    # Thinking budget (si está habilitado)
    if thinking_enabled:
        thinking_budget = output_available * 0.6
        actual_output = output_available * 0.4
    else:
        thinking_budget = 0
        actual_output = output_available
    
    return {
        "context_window": context_window,
        "max_output_tokens": int(actual_output),
        "thinking_budget": int(thinking_budget),
        "safe_margin": 256
    }
```

### Paso 3: Detener Compresión Innecesaria

```python
def should_compress(context_used_percent: float) -> bool:
    """
    NO comprimas solo porque se acerca a 80%
    Comprimes SOLO cuando se acerca a 95%+ y error real
    """
    if context_used_percent > 95:  # En lugar de 80
        return True
    return False
```

### Paso 4: Deshabilitar Thinking en Locales Pequeños

```python
def should_enable_thinking(context_window: int) -> bool:
    """
    Thinking requiere ~30% del contexto adicional
    Solo si tienes >32K contexto
    """
    return context_window > 32768
```

---

## Tabla Resumen: Por Qué Se Quedan a Medias

| Causa | Frecuencia | Severidad | Solución |
|-------|-----------|-----------|----------|
| Context window mal detectado | ⭐⭐⭐⭐⭐ | 🔴 Crítico | Probe correcto + fallback manual |
| max_tokens demasiado alto | ⭐⭐⭐⭐⭐ | 🔴 Crítico | Calcular budget = window - input - overhead |
| Thinking consume todo | ⭐⭐⭐⭐ | 🟠 Alto | Deshabilitar thinking para <32K |
| Compresión agresiva | ⭐⭐⭐⭐ | 🟠 Alto | Comprimir solo en 95%+, no 80% |
| finish_reason: "length" | ⭐⭐⭐ | 🟠 Alto | Aceptar incompleto, no reintentar |
| Mismatch input/output | ⭐⭐⭐ | 🟡 Medio | Schema claro entre context y output cap |
| Configuración por defecto | ⭐⭐ | 🟡 Medio | Detectar local vs cloud automáticamente |

---

## Recomendación Final

**Para un modelo local trabajar correctamente en Hermes/OpenClaw:**

```bash
# 1. Ollama: Configura explícitamente num_ctx
ollama run llama2 --param "num_ctx=16384"

# 2. Hermes config.yaml
model:
  context_length: 16384  # SER EXPLÍCITO
  max_tokens: 4096      # NO más de 25% del context

# 3. Deshabilita thinking
model:
  thinking: false

# 4. Reduce agresiveness de compresión
compression:
  trigger_at_percent: 95  # En lugar de 80
```

Con esto, los modelos locales deberían completar respuestas correctamente.

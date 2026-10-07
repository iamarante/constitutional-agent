# Constitutional AI System - Resumen Ejecutivo

## Pregunta Central
¿Cómo crear un sistema que obligue a los modelos a ser cautelosos **sin reentrenamiento**?

## Respuesta
No necesitas entrenar. Existen 6 capas de defensa que se pueden combinar **sin tocar los pesos del modelo**.

---

## Las 6 Capas de Defensa

### 1. **System Prompt Constitucional**
- Define principios éticos/legales
- El modelo los internaliza desde el inicio
- **Ventaja**: latencia cero, simple
- **Desventaja**: vulnerable a jailbreaking directo

```python
system_prompt = """
You follow these principles:
- Refuse illegal activities
- Be honest about limitations
- Protect user privacy
- Warn about dangerous code
"""
```

### 2. **Validación de Tool Calls**
- El servidor juzga ANTES de ejecutar cada herramienta
- Bloquea comandos peligrosos a nivel API
- **Ventaja**: previene acciones destructivas
- **Desventaja**: solo cubre herramientas, no respuestas textuales

```python
if tool_name == "shell" and dangerous_pattern in tool_input:
    return "Blocked: this action is not permitted"
```

### 3. **Post-Filter de Salida**
- Ejecuta un clasificador de seguridad en la respuesta final
- Detecta contenido unsafe y lo redacta/rechaza
- **Ventaja**: atrapa lo que el prompt y herramientas no bloquearon
- **Desventaja**: falsos positivos/negativos (~10-15%)

```python
if safety_classifier.is_unsafe(response):
    return "[Response filtered for safety]"
```

### 4. **Evaluador de Constitución**
- Un modelo pequeño (o el propio modelo) juzga si la salida respeta principios
- Segunda verificación independiente
- **Ventaja**: detecta evasiones sutiles
- **Desventaja**: +100-200ms latencia, requiere modelo evaluador

```python
for principle in constitution:
    if not evaluator.check(principle, response):
        return "Violates: " + principle
```

### 5. **Redacted Thinking**
- Filtra el pensamiento interno del modelo
- Solo expone razonamiento "seguro"
- **Ventaja**: evita que el usuario vea planes maliciosos
- **Desventaja**: complejidad, requiere extended thinking

```python
if block.type == "thinking":
    block.thinking = redact_unsafe_sentences(block.thinking)
```

### 6. **Auditoría y Policy Gating**
- Registra TODO: prompts, acciones, decisiones
- Aplica políticas: "si risk_score > X, pide aprobación"
- **Ventaja**: detecta patrones de ataque, permite rollback
- **Desventaja**: overhead operativo

```python
audit_log.record({
    "user": user_id,
    "action": tool_name,
    "risk_score": 0.7,
    "approved": False,
    "reason": "violates_policy"
})
```

---

## Arquitectura de Defensa en Capas

```
┌─────────────────────────────────────────────────────────┐
│  USER REQUEST                                           │
└────────────────────────┬────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│ CAPA 1: System Prompt Constitucional                   │
│ → Define principios básicos                            │
│ → El modelo intenta seguirlos                          │
└────────────────────────┬────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│ CAPA 2: Policy Gating (Risk Classification)            │
│ → ¿Es esta solicitud high-risk?                        │
│ → Si yes → pedir aprobación adicional                  │
└────────────────────────┬────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│ CAPA 3: Tool Call Validation                           │
│ → Bloquea herramientas/comandos peligrosos             │
│ → Server-side, no se puede eludir                      │
└────────────────────────┬────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│ CAPA 4: Model Reasoning + Tool Execution               │
│ → El modelo genera respuesta                           │
│ → Ejecuta herramientas validadas                       │
└────────────────────────┬────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│ CAPA 5: Post-Filter Output                             │
│ → Clasificador de seguridad en la salida               │
│ → Redacta/rechaza respuestas unsafe                    │
└────────────────────────┬────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│ CAPA 6: Constitutional Evaluation                      │
│ → Segundo modelo verifica principios                   │
│ → Chequeo final antes de devolver al usuario           │
└────────────────────────┬────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│ CAPA 7: Audit & Logging                                │
│ → Registra toda la cadena de decisiones                │
│ → Permite análisis y rollback                          │
└────────────────────────┬────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│  RESPONSE TO USER (Safe & Audited)                     │
└─────────────────────────────────────────────────────────┘
```

---

## Efectividad por Combinación

### Solo System Prompt
- **Efectividad**: ~60-70%
- **Riesgo**: Vulnerable a jailbreaking
- **Latencia**: Nula

### System Prompt + Tool Validation
- **Efectividad**: ~80-85%
- **Riesgo**: Medio (acciones bloqueadas, pero respuestas pueden ser malas)
- **Latencia**: Bajo

### System Prompt + Tool Validation + Post-Filter
- **Efectividad**: ~88-92%
- **Riesgo**: Bajo
- **Latencia**: Medio (+50ms)

### System Prompt + Tool Validation + Post-Filter + Constitutional Evaluator + Auditing
- **Efectividad**: ~94-97%
- **Riesgo**: Muy bajo
- **Latencia**: Alto (+200ms)
- **Costo**: Medio-Alto

---

## Diferencia: Constitutional AI vs Nuestro Sistema

### Constitutional AI (Anthropic - Con Entrenamiento)
- ✅ Cautela internalizada en los pesos del modelo
- ✅ Resistencia robusta a jailbreaking
- ✅ Razonamiento fundamentalmente seguro
- ✅ Latencia baja
- ❌ Requiere reentrenamiento ($$$$)
- ❌ No se puede cambiar dinámicamente

### Nuestro Sistema (Sin Entrenamiento)
- ✅ Sin reentrenamiento necesario
- ✅ Cambios dinámicos (actualizar principios en runtime)
- ✅ Funciona con cualquier modelo
- ✅ Muy auditable y transparent
- ❌ Latencia más alta (~200ms en caso completo)
- ❌ Vulnerable a jailbreaking si solo usas 1-2 capas
- ❌ Requiere más infraestructura

---

## Trade-offs

| Aspecto | Constitutional AI | Nuestro Sistema |
|--------|-------------------|------------------|
| Entrenamiento | Sí (costoso) | No |
| Latencia | ~0ms | ~50-200ms (depende capas) |
| Flexibilidad | Baja | Alta |
| Costo operativo | Bajo | Medio-Alto |
| Seguridad | 98-99% | 94-97% |
| Auditoría | Difícil | Fácil |
| Cambios dinámicos | No | Sí |

---

## Recomendación para Producción

**Nivel 1 (MVP - Trade-off latencia vs costo):**
- System Prompt Constitucional
- Tool Validation
- Post-Filter Básico
- Logging
- **Efectividad**: ~85-88%
- **Latencia**: ~50ms
- **Costo**: Bajo

**Nivel 2 (Estándar - Balanceado):**
- Todas las capas anteriores +
- Constitutional Evaluator
- Policy Gating avanzado
- **Efectividad**: ~92-95%
- **Latencia**: ~150ms
- **Costo**: Medio

**Nivel 3 (Enterprise - Máxima seguridad):**
- Todas las capas +
- Redacted Thinking
- Multi-model evaluation
- Human review para high-risk
- **Efectividad**: ~96-98%
- **Latencia**: ~300-500ms
- **Costo**: Alto

---

## Conclusión

No necesitas Constitutional AI de Anthropic para un sistema cauteloso.

Puedes construir uno casi igual de bueno combinando:
1. **Prompt coaching** (primero)
2. **Tool sandboxing** (crítico)
3. **Output filtering** (chequeo final)
4. **Evaluation loop** (validación)
5. **Auditing** (accountability)

El trade-off es latencia y complejidad operativa. Pero gainas flexibilidad, transparencia y la capacidad de actualizar políticas sin reentrenar.

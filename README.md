# Azure AI Foundry Agents – Cheat Sheet de Invocaciones REST y SDK

Guía rápida para el equipo sobre cómo **crear, invocar y orquestar agentes** de Azure AI Foundry usando  
1) **API REST** y 2) **SDK oficial para Python** (`azure-ai-agents`).  
Incluye integración con **Azure Function** como herramienta.

---

## 0. Convenciones

| Variable | Ejemplo | Descripción |
|----------|---------|-------------|
| `RESOURCE` | `proyecto-foundry` | Nombre del recurso Azure AI Foundry |
| `PROJECT`  | `agentes-proyecto` | Nombre del AI Project |
| `API_VER`  | `2025-05-15-preview` | Versión de la API (actual a jun-2025) |
| `ENDPOINT` | `https://$RESOURCE.services.ai.azure.com/api/projects/$PROJECT` |
| `KEY` / `TOKEN` | — | API Key (header `api-key`) **o** Bearer-token Entra ID |
| `AGENT_ID`, `THREAD_ID`, `RUN_ID` | — | IDs que devuelven las llamadas |

---

## 1. Autenticación

### REST
```http
# Header con API Key
api-key: <KEY>

# o Bearer token (Entra ID)
Authorization: Bearer <TOKEN>
```

### Python SDK
```python
from azure.identity import DefaultAzureCredential
cred = DefaultAzureCredential()        # usa Entra ID
ENDPOINT = "https://proyecto-foundry.services.ai.azure.com/api/projects/agentes-proyecto"
```

---

## 2. CRUD de Agentes

### 2.1 Crear agente – REST

```bash
curl -X POST "$ENDPOINT/agents?api-version=$API_VER" \
  -H "Content-Type: application/json" -H "api-key: $KEY" \
  --data '{
    "name": "agente_fiscal",
    "model": "gpt-4o",
    "instructions": "Eres un experto tributario para Perú.",
    "tools": [ { "type": "code_interpreter" } ]
}'
# ==> guarda AGENT_ID
```

### 2.2 Crear agente – Python

```python
from azure.ai.agents import AgentsClient
from azure.ai.agents.models import CodeInterpreterTool

client = AgentsClient(endpoint=ENDPOINT, credential=cred)
agent = client.create_agent(
    name="agente_fiscal",
    model="gpt-4o",
    instructions="Eres un experto tributario para Perú.",
    tools=[CodeInterpreterTool()]
)
AGENT_ID = agent.id
```

### 2.3 Actualizar

**REST:**
```bash
curl -X PATCH "$ENDPOINT/agents/$AGENT_ID?api-version=$API_VER" \
  -H "Content-Type: application/json" -H "api-key: $KEY" \
  --data '{ "instructions": "Responde en español y en ≤100 palabras." }'
```

**Python:**
```python
client.update_agent(AGENT_ID, instructions="Responde en español y en ≤100 palabras.")
```

### 2.4 Eliminar

**REST:**
```bash
curl -X DELETE "$ENDPOINT/agents/$AGENT_ID?api-version=$API_VER" \
  -H "api-key: $KEY"
```

**Python:**
```python
client.delete_agent(AGENT_ID)
```

---

## 3. Ejecutar conversación

### REST end-to-end

```bash
# 1) crear hilo
THREAD_ID=$(curl -s -X POST "$ENDPOINT/threads?api-version=$API_VER" \
  -H "api-key:$KEY" -d '{}' \
  | jq -r '.id')

# 2) mensaje de usuario
curl -X POST "$ENDPOINT/threads/$THREAD_ID/messages?api-version=$API_VER" \
  -H "api-key:$KEY" \
  --data '{ "role":"user", "content":"¿Cuál es la tasa IGV?" }'

# 3) run
RUN_ID=$(curl -s -X POST "$ENDPOINT/threads/$THREAD_ID/runs?api-version=$API_VER" \
  -H "api-key:$KEY" -d "{ \"agent_id\": \"$AGENT_ID\" }" | jq -r '.id')

# 4) polling hasta status == completed
curl "$ENDPOINT/threads/$THREAD_ID/runs/$RUN_ID?api-version=$API_VER" -H "api-key:$KEY"
```

### Python simplificado

```python
thread = client.threads.create()
client.messages.create(thread.id, role="user", content="¿Cuál es la tasa IGV?")
run = client.runs.create_and_process(thread.id, agent_id=AGENT_ID)
answer = client.messages.list(thread.id)[-1].content
print(answer)
```

---

## 4. Agregar una Azure Function como herramienta

### 4.1 Definición del agente (REST)

```json
{
  "name": "agente_pedidos",
  "model": "gpt-4o",
  "instructions": "Llama a get_customer_order cuando el usuario pregunte sobre su pedido.",
  "tools": [
    {
      "type": "azure_function",
      "azure_function": {
        "function": {
          "name": "get_customer_order",
          "description": "Estado del pedido activo.",
          "parameters": {
            "type": "object",
            "properties": { "customer_id": { "type": "string" } },
            "required": ["customer_id"]
          }
        },
        "input_binding": {
          "type": "storage_queue",
          "storage_queue": {
            "queue_service_uri": "https://storage.queue.core.windows.net",
            "queue_name": "aiagent-input"
          }
        },
        "output_binding": {
          "type": "storage_queue",
          "storage_queue": {
            "queue_service_uri": "https://storage.queue.core.windows.net",
            "queue_name": "aiagent-output"
          }
        }
      }
    }
  ]
}
```

### Crear agente

```bash
curl -X POST "$ENDPOINT/agents?api-version=$API_VER" \
  -H "Content-Type: application/json" -H "api-key:$KEY" \
  --data @agente_pedidos.json
```

### 4.2 Ejemplo de invocación con variable UI

```bash
# crear thread con metadata (token UI)
curl -X POST "$ENDPOINT/threads?api-version=$API_VER" \
  -H "api-key:$KEY" \
  --data '{ "metadata": { "customer_id": "CUST-123", "X-UI-Token": "tk-abc" } }'
```

El agente leerá `customer_id` de `thread.metadata` y lo pasará como argumento al ejecutar la función.

---

## 5. Llamar API externa (MCP) vía tool OpenAPI

```json
"tools": [
  {
    "type": "openapi",
    "openapi": {
      "specUrl": "https://api.midominio.com/mcp/openapi.yaml",
      "auth": {
        "type": "azure_key",
        "key": { "name": "x-api-key", "location": "header", "value": "<KEY>" }
      }
    }
  }
]
```

---

## 6. Orquestador de agentes

Declarar subagentes como herramientas tipo agent:

```json
"tools": [
  { "type": "agent", "agent": { "agent_id": "AGENTE_FISCAL_ID", "name": "fiscal" } },
  { "type": "agent", "agent": { "agent_id": "AGENTE_SOPORTE_ID", "name": "soporte" } }
]
```

En las instrucciones del orquestador define reglas de delegación.

---

## 7. Trazabilidad

```bash
curl -X GET "$ENDPOINT/threads/$THREAD_ID/runs/$RUN_ID/steps?api-version=$API_VER" \
  -H "api-key: $KEY"
```

Devuelve cada tool call, parámetros y respuestas.

---

## 8. Buenas prácticas proyecto

1. **Versionar** cada `*.json` de agentes y herramientas en Git.
2. **CI/CD** para aplicar POST/PATCH en Dev → QA → Prod.
3. **Habilitar Application Insights** para logs de runs y latencia.
4. **Usar RBAC**: rol Azure AI User para operación; Contributor solo para despliegues.
5. **Pasar datos sensibles** (tokens, IDs) en `thread.metadata`, no en texto de usuario.

---

**Para más detalles, consultar la doc oficial:**  
https://learn.microsoft.com/azure/ai-services/agents

---



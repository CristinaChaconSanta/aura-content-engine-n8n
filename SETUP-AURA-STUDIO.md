# Aura Studio - Content Engine | Guia de Setup

## Arquitectura Visual del Workflow (45 nodos)

```
ENTRADA (2 triggers → 1 router)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[Telegram Trigger] ──→ [Router Comandos] (Switch 7 salidas)
[Schedule Dom 8PM] → [Bridge] ──↗

RAMA EXTRAER (12 nodos)
━━━━━━━━━━━━━━━━━━━━━━
Router → TG "Extrayendo..."
            ├──→ [RSS URLs] → [HTTP Fetch x6] → [Parsear RSS] ──→ [Merge] → [Filter+ContextGraph+RAG]
            └──→ [GS Read Articulos] ────────────────────────────↗
                                                                     ↓
                                                              [IF hay nuevos?]
                                                              ├── SI → [AI Score] → [Guardrails] → [GS Save] → [Update Estado] → [TG Enviar Articulo]
                                                              └── NO → [TG "No hay nuevos"]

RAMA SI - Generar Carrusel (8 nodos)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Router → [GS Read Estado] → [Validar] → [GS Read Articulo]
    → [Prep Context+RAG+Brand] → [TG "Generando..."]
    → [AI Generar Carrusel] → [Guardrails Carrusel]
    → [GS Save Carousel] → [TG Enviar Carrusel]

RAMA NO - Descartar (5 nodos)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Router → [GS Estado + GS Articulos] → [Merge] → [Procesar] → [GS Update] → [TG Next/None]

RAMA SKIP (5 nodos)
━━━━━━━━━━━━━━━━━━━
Router → [GS Estado + GS Articulos] → [Merge] → [Procesar] → [GS Update] → [TG Next/None]

RAMA LISTO - Guardar Final (5 nodos)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Router → [GS Read Estado] → [Procesar] → [GS Clear + GS Update Context Graph] → [TG "Guardado"]

RAMA REGENERAR (7 nodos)
━━━━━━━━━━━━━━━━━━━━━━━━
Router → [GS Estado + GS Articulos] → [Merge] → [Prep] → [TG "Regenerando"]
    → [AI Regen] → [Guardrails] → [GS Save] → [TG Enviar]

DEFAULT
━━━━━━━
Router → [TG "Comando no reconocido"]
```

## Paso 1: Crear Google Sheet

Crear UN spreadsheet con 3 hojas:

### Hoja "articulos"
| Columna | Tipo |
|---------|------|
| title | texto |
| link | texto (clave unica) |
| source | texto |
| pubDate | fecha |
| description | texto |
| score | numero |
| categoria | texto |
| resumen | texto |
| datos_clave | texto (separados por \|) |
| tipo_sugerido | texto |
| estado | texto (nuevo/pendiente/aprobado/descartado/usado) |
| extractedAt | fecha |

### Hoja "estado"
| Columna | Tipo |
|---------|------|
| chat_id | texto |
| articulo_actual_link | texto |
| articulo_actual_title | texto |
| carrusel_actual | texto (JSON) |
| fase | texto (idle/esperando_aprobacion/esperando_revision) |
| updated_at | fecha |

### Hoja "context_graph"
| Columna | Tipo |
|---------|------|
| fecha | fecha |
| articulo_title | texto |
| articulo_link | texto |
| slides_count | numero |
| hashtags | texto |
| plataforma | texto |
| cta | texto |

## Paso 2: Configurar Credenciales en n8n

### Telegram Bot
1. Hablar con @BotFather en Telegram
2. Crear bot: `/newbot`
3. Copiar el token
4. En n8n: Settings > Credentials > New > Telegram API
5. Pegar token

### Obtener tu Chat ID
1. Hablar con @userinfobot en Telegram
2. Te dice tu chat ID
3. Reemplazar `CONFIGURAR_TU_CHAT_ID` en el nodo "Schedule Bridge"

### Google Sheets OAuth
1. n8n: Settings > Credentials > New > Google Sheets OAuth2
2. Seguir instrucciones de Google Cloud Console
3. Dar permisos de lectura/escritura

### OpenAI API
1. n8n: Settings > Credentials > New > OpenAI
2. Pegar tu API key

## Paso 3: Reemplazar Placeholders

En el workflow, buscar y reemplazar:
- `SPREADSHEET_ID_AQUI` → Tu ID de Google Sheet (esta en la URL)
- `CONFIGURAR_TU_CHAT_ID` → Tu chat ID de Telegram
- Todas las credenciales `"id": "CONFIGURAR"` se asignan desde n8n UI

## Paso 4: Importar y Activar

1. n8n > Workflows > Import from File
2. Seleccionar `aura-studio-complete.json`
3. Asignar credenciales a cada nodo (n8n te pedira)
4. Activar el workflow (toggle arriba a la derecha)

## Como se protege la marca (Guardrails)

### Guardrails de Scoring (despues del AI que puntua articulos):
- Score forzado al rango 1-100 (no puede inventar scores imposibles)
- Categoria validada contra lista cerrada
- Si el AI no retorna JSON valido → fallback con scores por defecto
- Resumen minimo requerido

### Guardrails de Carrusel (despues del AI que genera contenido):
- **4-10 slides** requeridos (no mas, no menos)
- **Hook maximo 12 palabras** en portada
- **Prompt de imagen obligatorio** en cada slide
- **Palabras PROHIBIDAS** en prompts de imagen:
  - `robot`, `circuit`, `neon`, `cyberpunk`, `matrix`, `binary code`, `motherboard`, `generic office`
  - Si aparecen → auto-reemplazo por `natural elements`
- **CTA validado**: solo `comenta`, `guarda`, `comparte`
- **Hashtags obligatorios**: `#AuraStudio` y `#WorkFastLiveSlow` se inyectan si faltan
- **Caption minimo** 10 caracteres

### Guardrails de Regeneracion:
- Mismas validaciones que el carrusel original
- Temperature subida a 0.9 para forzar variacion creativa

## Orden Context Graph → RAG (por que importa)

El workflow implementa el orden recomendado:

1. **Context Graph PRIMERO**: El nodo "Filter + Context Graph + RAG" carga primero
   la memoria institucional (historial, restricciones, calendario editorial).
   Esto le dice al AI: "ya cubrimos X temas, no repetir, el tono es Y".

2. **RAG DESPUES**: Con ese contexto, carga la identidad de marca (estetica,
   tono, paleta, tipografia, lo que se debe evitar).

3. **AI recibe ambos**: El prompt del AI incluye primero el Context Graph
   y luego el RAG, en ese orden. Asi el AI tiene criterio antes de crear.

Si se invierte: el AI leeria la marca primero y podria generar contenido
on-brand pero repetitivo o fuera de contexto temporal.

## Costos estimados

| Servicio | Costo por ejecucion |
|----------|-------------------|
| OpenAI GPT-4o (scoring) | ~$0.02-0.05 |
| OpenAI GPT-4o (carrusel) | ~$0.03-0.08 |
| OpenAI GPT-4o (regenerar) | ~$0.03-0.08 |
| Google Sheets | Gratis |
| Telegram Bot | Gratis |
| NewsAPI (si se agrega) | Gratis (100 req/dia) |
| **Total por ciclo completo** | **~$0.08-0.21** |

## Futuras mejoras

1. **KIE AI para imagenes**: Agregar nodo HTTP Request despues de "Guardrails Carrusel"
   que envie los `prompt_imagen` de cada slide a KIE AI

2. **Logo overlay**: Agregar nodo de procesamiento de imagen que superponga
   el logo PNG de Aura Studio en esquina inferior derecha

3. **Publicacion LinkedIn**: Agregar rama post-LISTO que publique via LinkedIn API

4. **Apify Reddit**: Agregar fuente de Reddit como en el workflow anterior
   para tendencias mas amplias

5. **Context Graph desde Sheets**: El nodo "Filter + Context Graph + RAG"
   actualmente tiene el context graph hardcodeado. Conectar a la hoja
   "context_graph" para que lea el historial real.

6. **Vector Store RAG**: Reemplazar el RAG hardcodeado con un vector store
   real (Supabase, Pinecone) que contenga todos los documentos de marca.

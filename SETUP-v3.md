# Aura Studio v3 - Template Rendering Pipeline

## Que cambio vs v2

El AI ya NO genera imagenes completas con texto.
Ahora hay 3 agentes desacoplados:

```
AGENTE REDACTOR (GPT-4o)          AGENTE VISUAL (DALL-E)         RENDERING (Bannerbear)
Genera textos por capas:     →    Genera SOLO fondos:        →   Ensambla todo:
- layer_title_text                - Sin texto                     - Fondo + Titulo + Body
- layer_body_text                 - Sin tipografia                - Logo de Aura Studio
- layer_footer_text               - Solo fotografia               - Tipografia Kaneda Gothic
- background_image_prompt         - Naturaleza latina             - Colores de marca exactos
```

Esto GARANTIZA que el logo, la tipografia y los colores siempre sean correctos.

## Arquitectura (3 capas, 3 secciones)

```
SECCION 1: GENERACION (arriba en el canvas)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

CAPA INGESTA          CAPA CEREBRO                    CAPA PRODUCCION
┌──────────┐     ┌───────────────────┐     ┌────────────────────────────────────┐
│ Trigger   │     │ Context Graph     │     │ Guardrails → Separar Slides       │
│ 4x/semana │──→──│ (memoria) PRIMERO │──→──│ → DALL-E (solo fondos)            │
│           │     │ RAG (marca) DESP  │     │ → Compilar para Render            │
│ RSS ──→───│──→──│ Filtrar duplicados│     │ → [Bannerbear] PLACEHOLDER        │
│ Fetch     │     │ Agente Editor     │     │ → Guardar Memoria → TG Carrusel   │
│ Parsear   │     │ (selecciona+genera│     │                                    │
└──────────┘     └───────────────────┘     └────────────────────────────────────┘

SECCION 2: APROBACION (medio del canvas)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Telegram Trigger → Router → APROBAR / RECHAZAR / REGENERAR

SECCION 3: RECAP SEMANAL (abajo del canvas)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Trigger Viernes 5PM → Leer descartes → AI Curador → Guardrails → TG Recap
```

## Nodos totales: 45 funcionales + 5 sticky notes

### Flujo principal (15 nodos):
1. Fuentes RSS → 2. Fetch RSS → 3. Parsear RSS
4. Leer Context Graph → Merge
5. Filtrar + RAG → Hay Noticias?
6. Agente Editor → 7. Guardrails
8. Separar Slides → 9. Generar Fondos (DALL-E)
10. Compilar para Render → 11. Render Plantilla [PLACEHOLDER]
12. Guardar en Memoria → 13-14. Buffer Rechazados → 15. TG Carrusel

### Aprobacion (11 nodos):
Telegram Trigger → Router → 3 ramas (APROBAR/RECHAZAR/REGENERAR) + Default

### Recap (7 nodos):
Trigger Viernes → R1-R6 → TG Recap

## Paso 1: Google Sheet

Crear UN spreadsheet con UNA hoja llamada "memoria":

| Columna | Tipo | Descripcion |
|---------|------|-------------|
| topic | texto | Tema del articulo |
| date | fecha | Cuando se proceso |
| source | texto | MIT/Wired/CIO/etc |
| source_url | texto | URL del articulo |
| status | texto | pending / published / rejected |
| angle | texto | Angulo o hook usado |
| carrusel_json | texto | JSON completo del carrusel |

## Paso 2: Credenciales en n8n

### Telegram Bot
1. @BotFather en Telegram → /newbot → copiar token
2. @userinfobot → copiar tu chat ID
3. n8n: Settings > Credentials > Telegram API

### Google Sheets
1. n8n: Settings > Credentials > Google Sheets OAuth2
2. Autorizar acceso

### OpenAI
1. n8n: Settings > Credentials > OpenAI
2. Pegar API key

### OpenAI Bearer (para DALL-E imagenes)
1. n8n: Settings > Credentials > Header Auth
2. Name: Authorization
3. Value: Bearer sk-tu-api-key-aqui

### Bannerbear (cuando lo configures)
1. n8n: Settings > Credentials > Header Auth
2. Name: Authorization
3. Value: Bearer bb-tu-api-key-aqui
4. Habilitar el nodo "11. Render Plantilla [PLACEHOLDER]"

## Paso 3: Reemplazar placeholders

Buscar y reemplazar en el JSON antes de importar:

| Buscar | Reemplazar con |
|--------|----------------|
| SPREADSHEET_ID | El ID de tu Google Sheet (esta en la URL) |
| TU_CHAT_ID | Tu chat ID de Telegram |

## Paso 4: Importar

1. n8n > Workflows > Import from File
2. Seleccionar aura-studio-v3.json
3. Asignar credenciales a cada nodo
4. Activar

## Schema de salida del Agente Editor

Cada carrusel genera este JSON desacoplado:

```json
{
  "meta": {
    "topic": "IA Agentes Autonomos",
    "source": "MIT News AI",
    "source_url": "https://...",
    "urgency": "High"
  },
  "design_settings": {
    "template_id": "aura_carousel_v1",
    "primary_color": "#1B4332",
    "accent_color": "#D4A574",
    "background_color": "#1A1A2E",
    "font_title": "Kaneda Gothic Bold",
    "font_body": "Inter Regular",
    "logo_variant": "cream",
    "logo_position": "bottom_right"
  },
  "slides": [
    {
      "number": 1,
      "type": "hook",
      "layer_title_text": "TU COMPETENCIA YA USA IA AGENTES",
      "layer_body_text": "",
      "layer_footer_text": "AURA STUDIO | 2026",
      "background_image_prompt": "Cinematic editorial photography of Colombian coffee plantation aerial view, morning mist between mountains, deep greens cream beige gold tones, no text, no typography, no letters, 1080x1080, 8k"
    }
  ],
  "caption": "Los agentes de IA ya no son futuro...",
  "hashtags": ["#AuraStudio", "#WorkFastLiveSlow", "..."],
  "cta": "comenta",
  "rejected_articles": [...]
}
```

Nota: layer_title_text, layer_body_text y layer_footer_text se renderizan
con la plantilla (Bannerbear/Creatomate). El AI NUNCA pone texto en las imagenes.

## Guardrails implementados

| # | Validacion | Auto-fix |
|---|-----------|----------|
| G1 | 5-6 slides obligatorio | Recorta a 6 si excede |
| G2 | layer_title_text presente | Alerta si falta |
| G3 | Titulo max 10 palabras | Alerta |
| G4 | Body max 30 palabras | Alerta |
| G5 | Footer default "AURA STUDIO" | Auto-agrega si falta |
| G6 | "no text" en background prompts | Auto-agrega si falta |
| G7 | Sin palabras prohibidas en prompts | Auto-reemplaza por "tropical nature" |
| G8 | design_settings presente | Aplica defaults si falta |
| G9 | Hashtags #AuraStudio obligatorio | Auto-agrega |
| G10 | CTA valido (comenta/guarda/comparte) | Default "comenta" |

Palabras prohibidas en background prompts:
robot, circuit, neon, cyberpunk, matrix, motherboard, generic office

## Costos estimados por ejecucion

| Servicio | Costo |
|----------|-------|
| GPT-4o (editor) | ~$0.05 |
| DALL-E 3 (5-6 fondos) | ~$0.20-0.24 |
| Bannerbear (5-6 renders) | ~$0.05 (segun plan) |
| **Total por carrusel** | **~$0.30** |
| **Total semanal (4+1 recap)** | **~$1.50** |

## Opciones de Rendering API

### Bannerbear (recomendado para empezar)
- Precio: desde $49/mes
- API simple, buena documentacion
- Soporta templates con capas

### Creatomate
- Precio: desde $39/mes
- Soporta video tambien
- Buena integracion con n8n

### Figma API + plugin
- Gratis (con cuenta Figma)
- Mas complejo de configurar
- Maximo control sobre el diseno

### HTML/CSS → Screenshot (gratis)
- Usar n8n Code node con HTML template
- Capturar con servicio de screenshot (urlbox, screenshotapi)
- Mas trabajo inicial pero $0 de costo

## Proximos pasos

1. Importar workflow y configurar credenciales
2. Crear plantilla en Bannerbear/Creatomate con:
   - Capa "title" (Kaneda Gothic Bold, color primario)
   - Capa "body" (Inter Regular)
   - Capa "footer" (pequeno, esquina inferior)
   - Capa "background" (imagen de fondo, full bleed)
   - Capa "logo" (PNG Aura Studio, bottom right)
3. Copiar el template_id a design_settings
4. Habilitar nodo "11. Render Plantilla"
5. Probar con ejecucion manual

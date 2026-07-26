# Content Engine - Guia de Configuracion para n8n

## Arquitectura del Workflow

```
[Trigger Domingo 8PM]
        |
[Config de Marca]
        |
   +---------+---------+
   |         |         |
[Reddit]  [Influencers]  [Noticias]
   |         |         |
   +---------+---------+
        |
  [Merge + Consolidar]
        |
  [Context Graph]  <-- PRIMERO: memoria historica
        |
  [RAG Catalogo]   <-- SEGUNDO: documentos de conocimiento
        |
  [AI Estratega]   <-- GPT-4o analiza todo y planifica
        |
  [Guardrails]     <-- Validacion anti-alucinaciones
        |
   +----+----+
   |         |
 [OK]    [FALLO] --> Alerta
   |
[Generar Imagenes]  <-- OpenAI gpt-image-1 (o Gemini gratis)
   |
[Ensamblar Paquete]
   |
[Actualizar Context Graph]
   |
[Notificar]
```

## Paso 1: Importar el Workflow

1. Abrir n8n
2. Ir a Workflows > Import from File
3. Seleccionar `workflow-content-engine.json`
4. El workflow aparecera con todos los nodos

## Paso 2: Configurar Credenciales

Necesitas crear estas credenciales en n8n (Settings > Credentials):

### Apify API Token
- Tipo: Header Auth
- Header Name: `Authorization`
- Header Value: `Bearer tu_token_de_apify`
- Asignar a los nodos: "Apify Reddit Scraper", "Apify Scraper Influencers"

### NewsAPI Key
- Tipo: Query Auth
- Query Parameter Name: `apiKey`
- Query Parameter Value: `tu_newsapi_key`
- Obtener en: https://newsapi.org
- Asignar al nodo: "NewsAPI Noticias Industria"

### OpenAI API Key
- Tipo: OpenAI API (built-in de n8n)
- API Key: `tu_openai_api_key`
- Asignar al nodo: "AI Estratega de Contenido"
- Para imagenes, crear tambien Header Auth:
  - Header Name: `Authorization`
  - Header Value: `Bearer tu_openai_api_key`
  - Asignar al nodo: "OpenAI Generar Imagenes"

## Paso 3: Configurar la Marca

Editar el nodo **"Configuracion de Marca"** con los datos reales:

- `name`: Nombre de tu marca
- `industry`: Tu industria
- `search_queries`: Terminos de busqueda para Reddit
- `news_keywords`: Keywords para buscar noticias
- `influencer_urls`: URLs de blogs/sitios de influencers
- `brand_voice`: Tono, personalidad, colores, estilo visual
- `target_platforms`: Plataformas objetivo

## Paso 4: Configurar el Context Graph

El archivo `context-graph-template.json` es la plantilla inicial.

### Opciones de almacenamiento (elegir una):

**Opcion A - Supabase (Recomendado)**
1. Crear tabla `context_graph` en Supabase
2. Reemplazar el nodo "Cargar Context Graph" con nodo Supabase
3. Reemplazar el nodo "Actualizar Context Graph" con nodo Supabase

**Opcion B - Google Sheets**
1. Crear Google Sheet con las hojas: historial, decisiones, performance
2. Usar nodos Google Sheets para leer/escribir

**Opcion C - Archivo JSON local**
1. Guardar `context-graph-template.json` en una ruta accesible por n8n
2. Usar nodos Read/Write File

## Paso 5: RAG con Vector Store (Opcional pero recomendado)

Para un RAG real en lugar del catalogo hardcodeado:

1. Subir documentos (catalogo, guia de marca, FAQ) a un vector store
2. En n8n, usar los nodos de AI:
   - **Supabase Vector Store** o **Pinecone** o **Qdrant**
   - Conectar al nodo **AI Agent** con tool de "Vector Store Retrieval"
3. Reemplazar el nodo "Cargar RAG" con la retrieval del vector store

## Paso 6: Imagenes - OpenAI vs Gemini

### OpenAI gpt-image-1 (de pago, mas preciso)
- Ya configurado en el workflow
- Costo: ~$0.04-0.08 por imagen
- Para 7 posts con ~5 slides promedio = ~$1.40-2.80/semana

### Gemini (gratis, mas rapido)
- Descomentar el nodo "[ALT] Gemini Imagenes"
- Desconectar "OpenAI Generar Imagenes"
- Conectar "Preparar Prompts" -> "[ALT] Gemini Imagenes" -> "Compilar Imagenes"
- Requiere API key de Google AI Studio (gratis)

## Paso 7: Conectar a LinkedIn (Opcional)

Para publicar automaticamente en LinkedIn:
1. Agregar nodo HTTP Request despues de "Notificar Exito"
2. Usar LinkedIn API v2:
   - POST a `https://api.linkedin.com/v2/ugcPosts`
   - Requiere OAuth 2.0 de LinkedIn

**ADVERTENCIA**: No conectar a Meta (Instagram/Facebook) por el momento.
Meta esta bloqueando cuentas que usan automatizacion con IA.

## Notas sobre el orden Context Graph -> RAG

El consejo de tu contacto es correcto y esta implementado:

1. **Context Graph PRIMERO**: Le da al agente la "memoria" de que se ha hecho,
   que funciono, que no, y las reglas editoriales. Es como darle experiencia.

2. **RAG DESPUES**: Con el contexto ya cargado, cuando el agente consulta
   el catalogo de productos y la guia de marca, puede filtrar inteligentemente
   que informacion es relevante para ESTA semana especifica.

Si se invierte el orden, el RAG devuelve informacion "cruda" sin criterio
y el agente puede generar contenido repetitivo o fuera de contexto.

## Guardrails implementados

1. Confidence score minimo (40/100)
2. Verificacion de fuentes vinculadas
3. Estructura completa obligatoria
4. Validacion de slides en carousels (3-10 slides)
5. Limite de caracteres en captions
6. Filtro de palabras prohibidas
7. Si nada pasa validacion -> alerta y no se generan imagenes

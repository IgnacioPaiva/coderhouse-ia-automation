# Curso IA Automation Specialist — Coderhouse

**Alumno:** Ignacio Paiva

Proyecto integrador único: **TiendaVerde**, un sistema agéntico de soporte y ventas para e-commerce. El workflow crece módulo a módulo; cada checkpoint parte del anterior y suma una capa nueva.

## Arquitectura actual

Chat y casilla de correo → filtro anti auto-reply → memoria persistente por sesión → router de triaje con taxonomía cerrada → manager que delega en workers especialistas vía sub-workflows → base de conocimiento documental consultada como herramienta → summarization automática → integraciones con CRM, Slack y Gmail → log de trazabilidad.

## Entregas

| Módulo | Capa que suma | Archivos |
|---|---|---|
| **M1** | Agente base en modo Tools Agent: system prompt modular, guardrail de 8 iteraciones, una tool nativa y log de observabilidad | [`M1/`](M1/) |
| **M2** | Orquestación multi-agente Manager-Worker con sub-workflows, contratos de datos JSON y vía de escape a supervisor humano | [`M2/`](M2/) |
| **M3** | Memoria persistente en Airtable por Session_ID, inyección de contexto con delimitadores y summarization automática a JSON estructurado | [`M3/`](M3/) |
| **M4** | Integraciones reales con CRM, Gmail y Slack: filtro anti auto-reply, look up previo al alta para evitar duplicados, borrador con aprobación humana y limpieza de payload | [`M4/`](M4/) |
| **M5** | Base de conocimiento documental (RAG): parseo con LlamaParse, vectorización con embeddings de Gemini, recuperación como herramienta del agente redactor, citación de fuentes y regla de contingencia | [`M5/`](M5/) |
| M6 | Capa de voz (STT / TTS) | _pendiente_ |
| **M7** | Diseño arquitectónico de un sistema agéntico vertical de industria — documento de consultoría, sin implementación | [`M7/`](M7/) |
| M8 | Supervisor AI-as-a-Judge y dashboard de calidad | _pendiente_ |
| **M9** | Gobernanza, costos y monitoreo del sistema en producción — documento de consultoría, sin implementación | [`M9/`](M9/) |

## Stack

n8n self-hosted (Docker) · OpenRouter · Google Gemini (chat y embeddings) · LlamaParse (LlamaCloud) · Airtable · Google Sheets (Service Account) · Gmail (OAuth2) · HubSpot · Slack

## Notas de implementación

**Autenticación.** Slack y HubSpot están autenticados mediante token de aplicación con scopes mínimos (`chat:write`, `channels:read`, `crm.objects.contacts.read/write`) por una restricción de la infraestructura self-hosted: el redirect URI de OAuth2 exige HTTPS estable, del que no se dispone con un túnel efímero. Gmail utiliza OAuth2 con callback en localhost.

**Modelo del agente redactor.** El redactor usa Google Gemini Flash y no un modelo gratuito de OpenRouter. La razón es concreta: varios modelos de capa gratuita no soportan *function calling* nativo y emiten la invocación de herramienta como texto plano (`<tool_call>`), con lo cual n8n nunca la interpreta como una llamada real y el agente entra en bucle hasta agotar `maxIterations`. Un Tools Agent sin function calling no falla con un error: responde sin usar las herramientas.

**Alcance de la base vectorial.** El nodo Simple Vector Store de n8n almacena bajo la clave de memoria **prefijada con el identificador del workflow**. Dos workflows distintos con la misma Memory Key no comparten datos. Por eso la ingesta de la base documental vive como rama independiente dentro del mismo workflow que la consume, y no como workflow separado.

**Limitaciones declaradas del RAG.** La base reside en memoria del proceso y se pierde al reiniciar n8n; el Simple Vector Store no expone umbral de similitud, por lo que el control de calidad de la recuperación está delegado al system prompt; y la reindexación es manual. Las tres están documentadas con su vía de resolución en el informe del M5.

## Cómo importar

n8n → menú ⋮ → **Import from File** → seleccionar el `.json`. Requiere configurar credenciales propias: las claves no viajan en la exportación de n8n. En los módulos con sub-workflows, importar primero los workers y después el manager.

Para el M5, ejecutar una vez el workflow de ingesta antes de consultar: sin ese paso el agente recupera cero fragmentos y responde correctamente que no dispone del dato.

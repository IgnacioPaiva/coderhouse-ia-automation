# Curso IA Automation Specialist — Coderhouse
**Alumno:** Ignacio Paiva

Proyecto integrador único: **TiendaVerde**, un sistema agéntico de soporte y ventas
para e-commerce. El workflow crece módulo a módulo; cada checkpoint parte del anterior
y suma una capa nueva.

## Arquitectura actual

Chat y casilla de correo → filtro anti auto-reply → memoria persistente por sesión →
router de triaje con taxonomía cerrada → manager que delega en workers especialistas
vía sub-workflows → summarization automática → integraciones con CRM, Slack y Gmail →
log de trazabilidad.

## Entregas

| Módulo | Capa que suma | Archivos |
|---|---|---|
| **M1** | Agente base en modo Tools Agent: system prompt modular, guardrail de 8 iteraciones, una tool nativa y log de observabilidad | [`M1/`](M1/) |
| **M2** | Orquestación multi-agente Manager-Worker con sub-workflows, contratos de datos JSON y vía de escape a supervisor humano | [`M2/`](M2/) |
| **M3** | Memoria persistente en Airtable por Session_ID, inyección de contexto con delimitadores y summarization automática a JSON estructurado | [`M3/`](M3/) |
| **M4** | Integraciones reales con CRM, Gmail y Slack: filtro anti auto-reply, look up previo al alta para evitar duplicados, borrador con aprobación humana y limpieza de payload | [`M4/`](M4/) |
| M5 | Base de conocimiento documental (RAG) | _pendiente_ |
| M6 | Capa de voz (STT / TTS) | _pendiente_ |
| M8 | Supervisor AI-as-a-Judge y dashboard de calidad | _pendiente_ |

## Stack

n8n self-hosted (Docker) · OpenRouter · Airtable · Google Sheets (Service Account) ·
Gmail (OAuth2) · HubSpot · Slack

## Notas de implementación

Slack y HubSpot están autenticados mediante token de aplicación con scopes mínimos
(`chat:write`, `channels:read`, `crm.objects.contacts.read/write`) por una restricción
de la infraestructura self-hosted: el redirect URI de OAuth2 exige HTTPS estable, del
que no se dispone con un túnel efímero. Gmail utiliza OAuth2 con callback en localhost.

## Cómo importar

n8n → menú ⋮ → **Import from File** → seleccionar el `.json`.
Requiere configurar credenciales propias: las claves no viajan en la exportación de n8n.
En los módulos con sub-workflows, importar primero los workers y después el manager.

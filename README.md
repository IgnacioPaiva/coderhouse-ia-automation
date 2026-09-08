# Curso IA Automation Specialist — Coderhouse
**Alumno:** Ignacio Paiva

Proyecto integrador único: **TiendaVerde**, un sistema agéntico de soporte y ventas
para e-commerce. El workflow crece módulo a módulo; cada checkpoint parte del anterior
y suma una capa nueva.

## Arquitectura actual

Chat → Router de triaje (taxonomía cerrada) → Manager que delega en workers
especialistas vía sub-workflows → log de trazabilidad.

## Entregas

| Módulo | Capa que suma | Archivos |
|---|---|---|
| **M1** | Agente base en modo Tools Agent: system prompt modular, guardrail de 8 iteraciones, una tool nativa y log de observabilidad | [`M1/checkpoint1_ignacio_paiva.json`](M1/) |
| **M2** | Orquestación multi-agente Manager-Worker con sub-workflows, contratos de datos JSON y vía de escape a supervisor humano | [`M2/`](M2/) — manager + 2 workers |
| M3 | Memoria persistente por Session_ID | _pendiente_ |
| M4 | Integraciones reales vía OAuth2 (CRM, Gmail, Slack) | _pendiente_ |
| M5 | Base de conocimiento documental (RAG) | _pendiente_ |
| M6 | Capa de voz (STT / TTS) | _pendiente_ |
| M8 | Supervisor AI-as-a-Judge y dashboard de calidad | _pendiente_ |

## Stack

n8n self-hosted (Docker) · OpenRouter · Google Sheets (Service Account) · Gmail (OAuth2)

## Cómo importar

n8n → menú ⋮ → **Import from File** → seleccionar el `.json`.
Requiere configurar credenciales propias: las claves no viajan en la exportación de n8n.
En el M2, importar primero los dos workers y después el manager.

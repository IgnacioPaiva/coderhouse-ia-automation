# Curso IA Automation Specialist — Coderhouse

**Alumno:** Ignacio Paiva · **Proyecto final:** aprobado (95 %) · Cohorte 2026

Proyecto integrador único: **TiendaVerde**, un sistema agéntico de soporte y ventas para e-commerce. El workflow crece módulo a módulo; cada checkpoint parte del anterior y suma una capa nueva.

## Arquitectura final

Webhook asíncrono (acuse HTTP 202 + `trace_id`), chat y casilla de correo → filtro anti auto-reply → memoria persistente por sesión → router de triaje con taxonomía cerrada → manager que delega en workers especialistas vía sub-workflows → base de conocimiento documental consultada como herramienta → supervisor automático de calidad que audita la respuesta antes de entregarla → summarization automática → integraciones con CRM, Slack y Gmail → devolución por callback al canal de origen → log de auditoría y trazabilidad → workflow de alertas ante cualquier falla.

## Entregas

| Módulo | Capa que suma | Archivos |
|---|---|---|
| **M1** | Agente base en modo Tools Agent: system prompt modular, guardrail de 8 iteraciones, una tool nativa y log de observabilidad | [`M1/`](M1/) |
| **M2** | Orquestación multi-agente Manager-Worker con sub-workflows, contratos de datos JSON y vía de escape a supervisor humano | [`M2/`](M2/) |
| **M3** | Memoria persistente en Airtable por Session_ID, inyección de contexto con delimitadores y summarization automática a JSON estructurado | [`M3/`](M3/) |
| **M4** | Integraciones reales con CRM, Gmail y Slack: filtro anti auto-reply, look up previo al alta para evitar duplicados, borrador con aprobación humana y limpieza de payload | [`M4/`](M4/) |
| **M5** | Base de conocimiento documental (RAG): parseo con LlamaParse, vectorización con embeddings de Gemini, recuperación como herramienta del agente redactor, citación de fuentes y regla de contingencia | [`M5/`](M5/) |
| **M6** | Capa de voz (STT / TTS): nota de voz por Telegram, transcripción, agente con base documental, síntesis y respuesta hablada. Circuito cerrado **operativo con costo cero** | [`M6/`](M6/) |
| **M7** | Diseño arquitectónico de un sistema agéntico vertical de industria — documento de consultoría, sin implementación | [`M7/`](M7/) |
| **M8** | Supervisor AI-as-a-Judge: segundo agente que audita cada respuesta contra sus fuentes, con métricas separadas de recuperación y generación, tres rutas de veredicto y medición comparativa A/B de arquitecturas | [`M8/`](M8/) |
| **M9** | Gobernanza, costos y monitoreo del sistema en producción — documento de consultoría, sin implementación | [`M9/`](M9/) |
| **M10** | Caso de negocio y propuesta comercial: horas liberadas, TCO, retorno y período de recupero | [`M10/`](M10/) |
| **PF** | **Proyecto Final Integrador — Documentación Maestra del Sistema Agéntico (DMSA):** entrada asíncrona por webhook con `trace_id` de punta a punta, supervisor AI-as-a-Judge antes de memoria y CRM, despacho por callback, workflow de alertas de falla, auditoría de 14 veredictos y justificación financiera | [`PF/`](PF/) |

## Stack

n8n self-hosted (Docker) · OpenRouter · Google Gemini (chat y embeddings) · Groq (Whisper Large v3) · ElevenLabs (TTS) · Telegram Bot API · LlamaParse (LlamaCloud) · Airtable · Google Sheets (Service Account) · Gmail (OAuth2) · HubSpot · Slack

## Notas de implementación

**Autenticación.** Slack y HubSpot están autenticados mediante token de aplicación con scopes mínimos (`chat:write`, `channels:read`, `crm.objects.contacts.read/write`) por una restricción de la infraestructura self-hosted: el redirect URI de OAuth2 exige HTTPS estable, del que no se dispone con un túnel efímero. Gmail utiliza OAuth2 con callback en localhost.

**Modelo del agente redactor.** El redactor usa Google Gemini Flash y no un modelo gratuito de OpenRouter. La razón es concreta: varios modelos de capa gratuita no soportan *function calling* nativo y emiten la invocación de herramienta como texto plano (`<tool_call>`), con lo cual n8n nunca la interpreta como una llamada real y el agente entra en bucle hasta agotar `maxIterations`. Un Tools Agent sin function calling no falla con un error: responde sin usar las herramientas.

**Alcance de la base vectorial.** El nodo Simple Vector Store de n8n almacena bajo la clave de memoria **prefijada con el identificador del workflow**. Dos workflows distintos con la misma Memory Key no comparten datos. Por eso la ingesta de la base documental vive como rama independiente dentro del mismo workflow que la consume, y no como workflow separado.

**Separación de modelos por rol.** El router de triaje, el agente redactor y el juez supervisor corren sobre modelos distintos, por dos razones independientes. Metodológica: un juez del mismo modelo que el agente evaluado incurre en sesgo de autopreferencia. Operativa: la capa gratuita de Gemini limita a **20 solicitudes por día y por modelo**, de modo que concentrar los tres roles en un solo modelo agota la cuota antes de terminar una batería de diez ejecuciones.

**Contexto del evaluador.** Los fragmentos que el agente recupera viven dentro de su bucle de razonamiento y se descartan al terminar la ejecución. Para que el juez pueda auditar contra las fuentes reales hay que activar la devolución de pasos intermedios en el agente y extraer de ahí únicamente las observaciones: el volcado crudo incluye la maquinaria interna del framework y desplaza al texto útil fuera de cualquier límite razonable de longitud. Un juez mal alimentado produce veredictos negativos indistinguibles de fallas reales del sistema evaluado.

**Audio: dos normalizaciones que no están documentadas en ningún lado.** Telegram entrega las notas de voz como `.oga`; Groq valida la extensión contra una lista cerrada que incluye `ogg` pero no `oga`, aunque el contenedor es el mismo, así que hay que renombrar el binario antes de enviarlo. Y en el sentido inverso, el nodo de ElevenLabs rotula su salida como `audio.textToSpeech.mp3` con mime `audio/mp3` **sin importar el Output Format elegido**: con OPUS seleccionado los bytes son Opus pero la etiqueta miente. Corregir ese mime a `audio/ogg` es suficiente para que Telegram muestre una nota de voz reproducible con el nodo nativo — **no hace falta `sendVoice`**, que n8n no implementa y que obligaría a exponer el token del bot en la URL de un HTTP Request.

**Entrada asíncrona y correlación.** El webhook responde `202 Accepted` desde un nodo propio apenas recibe la consulta y devuelve un `trace_id` (`trc_<fecha>_<id de ejecución>`). El mismo identificador queda en el log del juez, en las alertas de Slack y en la ejecución de n8n, así que una sola clave reconstruye todo el recorrido. Como se arma con el id de ejecución, sobrevive a la pausa del human-in-the-loop.

**Insertar un nodo en el medio cambia lo que reciben los de abajo.** Al poner el juez entre la consolidación y la persistencia de memoria, los nodos que leían `$json` pasaron a recibir la fila del log en lugar de los datos de la sesión: el resumen de Airtable se sobrescribía vacío sin ningún error visible. La corrección fue un nodo que reconstruye explícitamente el contexto de la sesión antes de persistir. Regla práctica: aguas abajo de un punto de inserción, referenciar nodos por nombre y no `$json`.

**Índice en memoria = incidente real.** Tras un reinicio de n8n el índice quedó vacío, el redactor respondió de memoria un plazo equivocado y el juez lo bloqueó por falta de fuentes. Es la mejor evidencia de para qué sirve el supervisor, y también la razón para pasar a un vector store persistente.

**Limitaciones declaradas del RAG.** La base reside en memoria del proceso y se pierde al reiniciar n8n; el Simple Vector Store no expone umbral de similitud, por lo que el control de calidad de la recuperación está delegado al system prompt; y la reindexación es manual. Las tres están documentadas con su vía de resolución en el informe del M5.

## Cómo importar

n8n → menú ⋮ → **Import from File** → seleccionar el `.json`. Requiere configurar credenciales propias: las claves no viajan en la exportación de n8n. En los módulos con sub-workflows, importar primero los workers y después el manager.

Para el M5, ejecutar una vez el workflow de ingesta antes de consultar: sin ese paso el agente recupera cero fragmentos y responde correctamente que no dispone del dato.

Para el proyecto final, importar `PF/PF_TiendaVerde - Alertas de Falla.json`, después el manager `PF/PF_M4 - ... Ev M8 juez.json`, y en Settings del manager elegir *TiendaVerde - Alertas de Falla* como **Error Workflow**. Prueba local de la entrada asíncrona:

```bash
curl -X POST http://localhost:5678/webhook-test/tiendaverde-consulta \
  -H 'Content-Type: application/json' \
  -d '{"sessionId":"demo-01","chatInput":"¿Cuánto tardan en mandar a Córdoba?","callback_url":"<opcional>"}'
```

Para el M8, importar primero el sub-workflow del juez y después pegar los nodos del supervisor en el manager: el nodo de ejecución necesita que el sub-workflow exista para poder seleccionarlo.

## Próximos pasos (devolución del proyecto final)

- Re-evaluar con el juez la respuesta corregida (máximo 1 iteración) y derivar a revisión humana si no alcanza 4/5; guardar versión original, crítica y corregida.
- Dataset de evaluación versionado de 50+ casos, separando ajuste y medición; medir falsos aceptados, falsos rechazados y errores críticos.
- Pruebas pendientes: callback externo, sesión de más de 5 intercambios, recuperación tras reinicio, timeouts y duplicados; latencia p50/p95.
- Endurecimiento para datos reales: acceso restringido a variables de entorno, roles mínimos, rotación de credenciales y retención/supresión de datos personales en memoria y logs.

# AI Automation · Proyecto integrador (Eliseo Sosa)

Repositorio de entregas del curso. Es un único proyecto que crece módulo a módulo: cada checkpoint parte del workflow anterior y le suma una capa.

| Archivo | Checkpoint | Qué es |
|---|---|---|
| `checkpoint1_eliseo_sosa.json` | M1 · Agente base | Agente calificador de leads con Google Docs como herramienta y log por Gmail |
| `checkpoint4_eliseo_sosa.json` | M4 · Integraciones reales | Manager-Worker con memoria (M3) conectado a Gmail, Pipedrive (CRM) y Slack por OAuth2 |

## Cómo importar cualquier workflow

1. En n8n: **Workflows › Add workflow › ⋯ › Import from File** y elegir el `.json`.
2. Abrir cada nodo marcado con un triángulo de advertencia y elegir la credencial propia (los IDs del JSON son de mi instancia; n8n pide reasignarlos).
3. Verificar que el semáforo de cada credencial quede en verde (**Credentials › Test**).
4. Ejecutar con **Execute Workflow**: cada workflow trae un caso de prueba en `pinData`, así que corre sin esperar un evento real.

Hecho para n8n 1.x (self-hosted, versión con AI Agent v3). Nodos de IA: `@n8n/n8n-nodes-langchain`.

---

## Checkpoint 4 · Sincronización del cerebro agéntico con el ecosistema de negocio

### Criterios de evaluación → evidencia

| Criterio | Peso | Dónde se cumple | Evidencia |
|---|---|---|---|
| Conectividad segura y mínimo privilegio | 40 % | 3 integraciones reales por OAuth2: Gmail (`gmailOAuth2`), Pipedrive (`pipedriveOAuth2Api`) y Slack (`slackOAuth2Api`). Scopes recortados: ver *Credenciales y scopes* | [credencial_gmail.png](evidencias/checkpoint4/credencial_gmail.png), [credencial_pipedrive.png](evidencias/checkpoint4/credencial_pipedrive.png), [credencial_slack.png](evidencias/checkpoint4/credencial_slack.png): las tres credenciales de tipo *OAuth2* en **Account connected**, en verde |
| Contención de bucles y mitigación de errores | 35 % | IF anti-bucle pegado al trigger (19 condiciones); Set + IF de email válido (400); Look up antes del Create (409); si el Look up falla, el workflow se detiene | [pipedrive_personas.png](evidencias/checkpoint4/pipedrive_personas.png): después de dos mails del mismo remitente hay **una sola** persona con ese email (la 2ª corrida fue por Update). La corrida anti-bucle está en la tabla de *Prueba de regresión* |
| Gobernanza operativa e interfaz HITL | 25 % | Única escritura de Gmail: `draft › create` en el mismo hilo. Ningún nodo envía correo | [gmail_borrador.png](evidencias/checkpoint4/gmail_borrador.png): la respuesta de la IA como borrador dentro del hilo del cliente, con *Enviar* pendiente de un humano |

Workflow completo ejecutado en verde: [workflow_verde.png](evidencias/checkpoint4/workflow_verde.png).

### Propósito

Es el mismo Manager-Worker con memoria de largo plazo del Módulo 3, pero ahora vive dentro del stack real de una tienda e-commerce:

- **Entrada:** la casilla de soporte en **Gmail** (antes era un webhook).
- **Cerebro:** Router de triaje → Worker Calificador (VENTAS) o Worker Soporte (SOPORTE) → memoria en Airtable con resumen condicional. Sin cambios de lógica respecto del M3.
- **Salida:** sincroniza el contacto en el CRM (**Pipedrive**), deja la respuesta de la IA como **borrador en Gmail** para que la apruebe una persona y avisa al canal de operaciones en **Slack**.

```
Gmail Trigger (casilla de soporte)
  │
  ▼
① IF ¿Es respuesta automática? ──TRUE──▶ Descartado: auto-reply (fin del bucle)
  │ FALSE
  ▼
Datos del lead → Memoria Airtable (M3) → Router → Workers → Resumen → Persistir memoria
  │
  ▼
④ Set Payload limpio y validado (CRM) → IF ¿Email válido? ──FALSE──┐
  │ TRUE                                                           │
  ▼                                                                │
② Pipedrive Buscar contacto (Look up)                             │
  │                                                                │
  ▼                                                                │
IF ¿El contacto ya existe?                                         │
  ├─ SÍ ─▶ Actualizar contacto en CRM (Update)                     │
  └─ NO ─▶ Crear contacto en CRM (Create)                          │
  │                                                                │
  ▼                                                                │
Registrar nota del caso en CRM                                     │
  │                                                                │
  ▼                                                                │
③ Gmail Crear borrador de respuesta (Create Draft · HITL)          │
  │                                                                │
  ▼                                                                ▼
⑤ Set Payload mínimo para Slack ◀──────────────────────────────────┘
  │
  ▼
Slack Aviso al canal de Operaciones
```

### Por qué Pipedrive y no HubSpot

La consigna pide *"el CRM de la tienda (HubSpot o Salesforce)"* y aclara en la sección de la clase en vivo: *"no necesitás las mismas apps… usás las que tengas a mano mientras respeten el mismo rol"*. Pipedrive cumple el rol de CRM (fuente única de verdad del cliente) con un conector nativo de n8n y OAuth2.

Se descartó HubSpot porque hoy ya no permite crear apps públicas OAuth desde la interfaz (solo por CLI con proyectos), y la app privada usa un token fijo, no OAuth2. El nodo de Pipedrive además tiene operaciones separadas de **Search**, **Create** y **Update**, que calzan uno a uno con el patrón Look up → Create/Update de la rúbrica.

### Requisito de la consigna → dónde está en el JSON

| Paso | Requisito | Nodo (`name` en el JSON) | Configuración clave |
|---|---|---|---|
| 1 | Tres conectores nativos: CRM, Gmail, Slack | `Buscar contacto en Pipedrive (Look up)`, `Crear contacto en CRM (Create)`, `Actualizar contacto en CRM (Update)`, `Registrar nota del caso en CRM` · `Casilla de soporte (Gmail Trigger)`, `Crear borrador de respuesta (Create Draft · HITL)` · `Aviso al canal de Operaciones (Slack)` | `n8n-nodes-base.pipedrive`, `gmailTrigger`, `gmail`, `slack` |
| 2 | Autenticación OAuth2 | Los mismos nodos | `pipedriveOAuth2Api`, `gmailOAuth2`, `slackOAuth2Api`; `authentication: "oAuth2"` en Pipedrive y Slack |
| 3 | Lectura (pasado) y escritura (futuro) con mínimo privilegio | Lectura: Gmail Trigger, Airtable Search, Pipedrive Search. Escritura: Airtable Update, Pipedrive Create/Update + nota, Gmail Draft, Slack post | Ver tabla de scopes abajo. Pipedrive busca solo en el campo `email` y escribe nombre + email (Create), solo nombre (Update) y una nota |
| 4 | IF inmediatamente después del trigger de correo | `¿Es respuesta automática? (anti-bucle)` | Conectado directo a la salida del Gmail Trigger |
| 5 | Ignorar Auto-reply, Out of office, Undeliverable, no-reply@ | `¿Es respuesta automática? (anti-bucle)` | 19 condiciones en modo **OR**, sin distinguir mayúsculas: asunto (`Auto-reply`, `Automatic reply`, `Respuesta automática`, `Out of office`, `Fuera de la oficina`, `Undeliverable`, `Delivery Status Notification`, `Mail delivery failed`), remitente (`no-reply`, `noreply`, `donotreply`, `mailer-daemon`, `postmaster`) y headers RFC 3834 (`Auto-Submitted: auto-*`, `Precedence: bulk/junk/auto_reply`, `X-Autoreply`). TRUE → `Descartado: auto-reply (fin del bucle)` |
| 6 | Look up antes del Create (Error 409) | `Buscar contacto en Pipedrive (Look up)` → `¿El contacto ya existe?` | `person › search`, `fields: email`, `exactMatch: true`, `limit: 1`, `alwaysOutputData: true` (sin resultado = item vacío). Si existe → `person › update` por ID; si no → `person › create`. Si la búsqueda falla, el workflow se detiene: un error nunca se interpreta como "no existe" |
| 7 | Salida de correo solo como Create Draft (HITL) | `Crear borrador de respuesta (Create Draft · HITL)` | `resource: draft`, `operation: create`, con `threadId` y `sendTo`. El workflow no tiene ningún nodo que envíe correo |
| 8 | Set / filtro antes de Slack | `Payload mínimo para Slack` | `includeOtherFields` apagado: sale un solo texto armado + 2 campos de estado. Sin cuerpo del mail, sin HTML, sin adjuntos |
| 8 bis | Set de limpieza y validación antes del CRM (Error 400) | `Payload limpio y validado (CRM)` → `¿Email válido? (evita Error 400)` | Deja 9 campos; valida el email con regex y corta si está vacío o mal formado |
| 9 | Test de regresión nodo por nodo | Ver "Prueba de regresión" | `pinData` del trigger con un email real de ejemplo |

### Los cuatro controles de la rúbrica, explicados

**① Anti-bucle.** Si la tienda responde a un "Fuera de la oficina" y ese servidor vuelve a responder, se arma un ping-pong infinito que quema tokens y ensucia el CRM. El IF es el primer nodo después del trigger: ningún mail automático llega al agente, al CRM ni a Slack. Además de los asuntos que pide la consigna, revisa los headers estándar de respuesta automática, que son más confiables que el asunto porque no dependen del idioma.

**② Look up antes del Create.** En HubSpot o Salesforce, crear un contacto con un email que ya existe devuelve **409 Conflict**. Pipedrive es peor: no da error y crea una segunda persona con el mismo email, así que el duplicado pasa en silencio. El workflow primero busca por email con coincidencia exacta y recién después decide. En la rama Update solo se toca el nombre, por ID; el email no se reescribe. En las dos ramas el estado del caso queda como **nota** en la ficha de la persona, así el historial se acumula en un solo contacto.

**③ Human-in-the-loop.** La respuesta de la IA nunca sale sola: queda en **Borradores** de Gmail, dentro del mismo hilo del cliente. Una persona la revisa, la edita si hace falta y la envía.

**④ Payload limpio.** Dos Set con `Include Other Fields` apagado, lo que en n8n descarta todo lo que no se nombra explícitamente, incluidos los binarios:
- antes del CRM: 9 campos tipados y un email validado (evita el **400 Bad Request** por email vacío o mal formado);
- antes de Slack: un mensaje de 6 líneas. El cuerpo del mail, el HTML y los adjuntos nunca llegan al canal.

A eso se suma que el Gmail Trigger tiene `downloadAttachments: false` y que `Datos del lead` convierte el mail a texto plano, corta el hilo citado y limita a 3000 caracteres antes de pasarlo al agente.

### Credenciales y scopes (mínimo privilegio)

| Herramienta | Credencial n8n | Operaciones que usa el workflow | Scopes mínimos | Cómo se configuran |
|---|---|---|---|---|
| Gmail (casilla de soporte) | Gmail OAuth2 API | Leer mensajes nuevos (trigger) y crear borradores | `https://www.googleapis.com/auth/gmail.readonly` y `https://www.googleapis.com/auth/gmail.compose` | En la credencial activar **Custom Scopes** y dejar solo esos dos. Sin `gmail.modify` ni `mail.google.com` (no puede borrar ni mover mails) |
| Pipedrive (CRM) | Pipedrive OAuth2 API | Buscar, crear y actualizar personas; crear notas | **Contacts: Full access** (personas y sus notas). Nada de deals, actividades, productos ni administración | App privada en el Developer Hub de una cuenta sandbox de Pipedrive, con la Callback URL de n8n y ese único scope marcado |
| Slack (canal de operaciones) | Slack OAuth2 API | Publicar un mensaje en `#operaciones` | `chat:write` (y `channels:read` si se elige el canal desde la lista) | App propia en api.slack.com con la Redirect URL de n8n. En la credencial activar **Custom Scopes** y dejar solo esos |
| Airtable (memoria, del M3) | Airtable Personal Access Token | Buscar, crear y actualizar la sesión | `data.records:read`, `data.records:write` sobre la base `Memoria Agentica` | Token limitado a esa única base |
| Google Gemini | Google Gemini (PaLM) API | Modelo del Router y del resumidor | API key | — |

Además del scope, el mínimo privilegio se aplica a nivel de campo: Pipedrive busca en 1 campo y escribe 1 o 2 más una nota; Slack recibe 3 campos; Airtable guarda solo hechos accionables (ver M3).

### Qué hay que reemplazar al importar

| Dónde | Valor en el JSON | Reemplazar por |
|---|---|---|
| 4 nodos Pipedrive | `REEMPLAZAR_CREDENCIAL_PIPEDRIVE` | Tu credencial Pipedrive OAuth2 |
| Nodo Slack | `REEMPLAZAR_CREDENCIAL_SLACK` y canal `#operaciones` | Tu credencial y tu canal |
| `Delegar a Worker Calificador` / `Delegar a Worker Soporte` | IDs de los sub-workflows del M3 | Los IDs de tus Workers importados |
| Airtable, Gmail, Gemini | IDs de mi instancia | Tus credenciales |

### Prueba de regresión

Ejecutada el 21/09/2026 con mails reales enviados a la casilla de soporte y traídos con **Fetch Test Event** del Gmail Trigger:

| # | Mail de entrada | Camino esperado | Resultado |
|---|---|---|---|
| 1 | Consulta comercial nueva ("Automatizar carga de pedidos de WhatsApp") | Anti-bucle FALSE → memoria → Router VENTAS → Worker Calificador → Look up sin resultado → **Create** → nota → **borrador** → Slack | OK: persona creada en Pipedrive con nota y borrador en el hilo |
| 2 | Seguimiento del mismo remitente ("Consulta adicional…") | Look up encuentra a la persona → **Update** (no Create) → segunda nota → borrador → Slack | OK: una sola persona con dos notas. Sin duplicado (el caso 409) |
| 3 | "Respuesta automática: Fuera de la oficina" | Anti-bucle TRUE → `Descartado: auto-reply (fin del bucle)` | OK: no se tocó Pipedrive, Gmail ni Slack |

Para repetirla: mandar un mail a la casilla conectada, abrir el Gmail Trigger, **Fetch Test Event** y **Execute Workflow**. El `pinData` del JSON trae un mail de ejemplo para recorrer el flujo sin esperar un correo; con ese pin el nodo del borrador da "not found", porque el `threadId` del ejemplo no existe en ninguna casilla real.

Nota sobre Slack: el nodo `Aviso al canal de Operaciones (Slack)` está configurado para **continuar si falla** (`onError: continueRegularOutput`). Es una decisión de diseño: el aviso es informativo, y un problema en Slack no debe deshacer ni frenar un caso que ya se registró en el CRM y ya dejó el borrador. La contracara es que ese nodo puede pintarse en verde aunque el mensaje no haya salido, así que el aviso se verifica mirando el canal.

### Cambios respecto del M3

- Webhook reemplazado por Gmail Trigger; `Datos del lead` conserva el mismo nombre y las mismas claves, así el resto del flujo no cambió.
- El log de auditoría por Gmail (`Log de trazabilidad`) se reemplazó por el aviso en Slack. Motivo: la consigna pide que la única escritura de Gmail sea Create Draft, y un mail enviado a la propia casilla volvería a entrar por el trigger. La traza completa queda en el historial de ejecuciones de n8n.
- `Respond to Webhook` se quitó porque ya no hay webhook: la respuesta al cliente es el borrador.
- Correcciones de la devolución del M3: se arregló la expresión de `contador_intercambios` que se enviaba al Worker Soporte, `Resumen sin cambios` conserva los puntos clave previos en vez de volcar el JSON de datos duros, y las claves quedan unificadas sin tilde (`accion_requerida` en el prompt, el parser, el Set y el campo `Accion_Requerida` de Airtable).
- Mapeo de nombres de memoria respecto de la consigna del M3: `user_name` = `memoria_usuario` (campo `Nombre_Usuario`), `last_summary` = `memoria_resumen` (campo `Resumen_Consolidado`).

### Modelo

Se usa **Google Gemini** (`gemini-3.1-flash-lite`) en el Router y en el resumidor. La consigna menciona GPT-4o o Claude como preferentes; la elección de Gemini es por costo (el Router y el resumidor son tareas de clasificación y compresión con salida estructurada, donde un modelo liviano alcanza) y porque es el que usan los workflows de clase. El cambio es directo: conectar un **OpenAI Chat Model** o **Anthropic Chat Model** al mismo puerto `ai_languageModel` del agente; los prompts y los parsers no dependen del proveedor.

---

## Checkpoint 1 · Agente calificador de leads

**Propósito:** recibe un lead por webhook, un AI Agent lo califica contra los criterios guardados en un Google Doc y responde. Deja un log de auditoría y una alerta de error por Gmail.

**Nodos:** `Entrada de lead` (Webhook) → `Datos del lead` (Set) → `AI Agent Calificador de Leads` (Tools Agent, `maxIterations: 6`, `returnIntermediateSteps` activo) con `Criterios de calificacion de leads` (Google Docs tool por el puerto `ai_tool`) y Gemini como modelo → `Log de observabilidad` (Gmail) y `Respond to Webhook`. La salida de error del agente va a `Alerta de error` (Gmail).

**Credenciales:** Google Docs OAuth2 (el agente solo lee el documento de criterios; en el nodo va el ID del documento, no la URL), Gmail OAuth2 (envío del log y la alerta) y Google Gemini API.

**Modo Tools Agent:** desde n8n 1.82 el nodo AI Agent funciona como Tools Agent por defecto, por eso el JSON no incluye el parámetro `agent: toolsAgent`.

**Prueba:** el webhook trae un caso fijado en `pinData`; **Execute Workflow** lo corre completo.

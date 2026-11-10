# Checkpoint 1 · Agente de Triaje de Briefs · Freebag

Primera versión del proyecto integrador del curso **AI Automation Avanzado (Coderhouse)**.

**Archivo entregable:** `checkpoint1_santos_tedin.json`

**Freebag** es un marketplace argentino que conecta marcas y pymes con talento creativo freelance. Este agente recibe pedidos escritos en lenguaje libre ("necesito fotos para la colección, es para el viernes"), los interpreta, decide si tiene datos suficientes y, si los tiene, registra el brief estructurado en Google Sheets para que el equipo asigne un freelancer.

## Arquitectura

```
[Chat Trigger] → [AI Agent · Tools Agent · Max Iterations 6] → [Gmail · Log de observabilidad] → [Respuesta al chat]
                        ├─ Chat Model: Google Gemini (gemini-3.5-flash-lite)
                        ├─ Memory: Simple Memory (10 mensajes por sesión)
                        └─ Tool: Registrar_Brief_Freebag (Google Sheets · Append Row)
```

| Requisito del checkpoint | Cómo se cumple |
|---|---|
| Trigger desestructurado | Chat Trigger nativo de n8n |
| AI Agent en modo Tools Agent | Nodo AI Agent (Tools Agent con function calling) |
| Chat Model nativo | Google Gemini Chat Model · temperatura 0.2 · Retry On Fail (3 intentos). Se eligió Gemini por costo cero en el plan gratuito y soporte de tool calling |
| Guardrail de iteraciones | Max Iterations = 6 |
| System Prompt modular | Rol → Ámbito → Objetivo → Reglas de herramientas → Escalamiento → Estilo, con acciones prohibidas explícitas |
| Tool lateral con descripción extensa | Google Sheets conectada en el puerto **Tool** del agente, nunca en secuencia |
| Observabilidad | Gmail con el mensaje recibido, las tools ejecutadas (intermediate steps), los datos enviados, el resultado y la respuesta final |

**Frontera lineal / agéntica:** el trigger, el log y la respuesta son deterministas. El agente solo se ocupa de la parte ambigua: interpretar el pedido, clasificarlo y decidir si registrar o preguntar.

## Guardrails

- **Max Iterations 6:** evita loops y gasto descontrolado de tokens.
- **Mínimo privilegio:** la tool solo puede **agregar filas** (Append Row). No lee, edita ni borra.
- **Human-in-the-loop:** la columna `revision humana` se marca "Sí" con presupuesto mayor a USD 3.000, cesión de derechos o exclusividad, reclamos o reembolsos.
- **Acciones prohibidas:** cotizar, prometer plazos, asignar freelancers, gestionar pagos, dar asesoramiento legal y borrar o modificar registros.

## Pruebas realizadas

| # | Mensaje | Resultado |
|---|---|---|
| A | "Hola, soy Juli de Mila Indumentaria (juli@milaind.com). Necesitamos fotos de producto para la colección de verano, son 40 prendas, para el 20/11. Presupuesto USD 800." | ✅ El agente ejecutó `Registrar_Brief_Freebag` por decisión propia: Fotografía · Media · USD 800 · 20/11 · Revisión humana: No. Fila registrada en la planilla y log recibido por Gmail. Flujo completo en verde |
| B | "Quiero un logo para mi cervecería" | ✅ El agente **no** usó la tool y pidió un medio de contacto, como indica el System Prompt |

## Otros mensajes para probar

| Mensaje | Resultado esperado |
|---|---|
| "Somos Altamar (contacto@altamar.com.ar). Queremos un spot para TV con cesión total de derechos, presupuesto USD 5.000." | Usa la tool · Producción audiovisual · Revisión humana: Sí |
| "¿Cuánto me cobrarían por 10 reels? Pasame precio ya." | No cotiza · deriva a una persona del equipo |
| "¿Qué hora es en Madrid?" | No usa la tool · aclara que solo gestiona briefs |

## Cómo importarlo

1. Crear una Google Sheet con estos encabezados en la fila 1: `Fecha, Nombre, Marca, Contacto, Categoria, Prioridad, Presupuesto, Plazo, Resumen, Proxima accion, revision humana`.
2. En n8n: **Import from File** → `checkpoint1_santos_tedin.json`.
3. Conectar credenciales propias en **Google Gemini Chat Model** (API key gratis de aistudio.google.com), **Registrar_Brief_Freebag** (Google Sheets OAuth2) y **Gmail - Log de Observabilidad** (Gmail OAuth2).
4. En la tool, elegir la planilla propia. En Gmail, cambiar el destinatario.
5. Abrir el **Chat** del workflow y probar.

> El JSON se exporta sin credenciales: cada persona conecta las suyas.

## Próximos módulos

M2 multi-agente (Manager + Workers) · M3 memoria por Session_ID · M4 CRM e integraciones · M5 RAG sobre los términos, tarifas y FAQ de Freebag · M6 voz.

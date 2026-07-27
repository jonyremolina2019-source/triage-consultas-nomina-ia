# Proyecto Final — Sistema de Triage Automatizado de Consultas de Nómina con IA

Automatización end-to-end para un estudio de liquidación de sueldos argentino: las consultas que los clientes envían por email se clasifican con IA, se genera un borrador de respuesta profesional, y un humano aprueba antes de que la respuesta llegue al cliente (Human-in-the-Loop).

> Proyecto final del curso de Automatización con IA — Ticher.

---

## 🧩 Tecnologías integradas (4/4)

| Rol | Tecnología | Función |
|---|---|---|
| Orquestador | **Make** | 2 escenarios: flujo principal de triage + webhook de aprobación (HITL) |
| Base de datos | **Airtable** | Memoria del sistema: 3 tablas relacionadas con campos de estado |
| Procesamiento IA | **Google Gemini (3.5 Flash)** | Clasificación de la consulta + generación del borrador, salida JSON estructurada |
| Canal de salida | **Gmail + Slack** | Gmail: entrada y respuesta en el mismo hilo (Thread ID). Slack: notificación de aprobación |

## 🏗️ Arquitectura

```
Cliente → Gmail (trigger From now on + Label)
        → Airtable: buscar cliente por email remitente
        │    └─ [no existe] → Log_Errores + aviso (camino infeliz)
        → Airtable: crear consulta (Estado: Pendiente)
        → Gemini: clasificar + borrador (JSON, tokens limitados)
        │    └─ [fallo API] → Error handler RESUME → Log_Errores (Estado: Error)
        → Airtable: update (Estado: Procesado por IA)
        → Slack: solicitud de aprobación ✅/❌  ← FIN Escenario 1 (HITL)

[Webhook aprobación] → Escenario 2
        → aprobar  → Gmail Reply en el Thread ID original → Estado: Enviado
        → rechazar → Estado: Rechazado + aviso Slack
```

Diagrama completo: ver `Diagrama_Arquitectura_Proyecto_Final.pdf`.

## 🧠 Modelo de datos (Airtable)

- **Clientes**: razón social, CUIT, email (clave de match), CCT aplicable, estado.
- **Consultas**: asunto, cuerpo, Thread ID de Gmail, categoría IA, urgencia, borrador, **Estado** (`Pendiente → Procesado por IA → Esperando Aprobación → Aprobado/Rechazado → Enviado | Error`), vínculo a Clientes.
- **Log_Errores**: módulo que falló, mensaje ({{error.message}}), timestamp automático, vínculo a Consultas.

Relaciones bidireccionales: Clientes ↔ Consultas ↔ Log_Errores.

🔗 **Base en modo lectura:** [https://airtable.com/invite/l?inviteId=inva8WLZXQU3AgFyH&inviteToken=0e7a30332efca0befdf939f45eb78558dff5d4707b44c327089ffd337f741980&utm_medium=email&utm_source=product_team&utm_content=transactional-alerts]

## 🤖 Prompt del motor IA

System prompt completo en [`prompt_ia.txt`](prompt_ia.txt). Puntos clave:
- Categorías y urgencias escritas EXACTO como las opciones de los single select de Airtable (evita rechazos de valor).
- Salida forzada a JSON de una sola línea, sin markdown ni saltos de línea reales (ver "Decisiones técnicas").
- Máx. 120 palabras de borrador + límite de output tokens → optimización de costos (1 crédito por operación).

## ⚠️ Resiliencia (gestión de errores)

| Punto de fallo | Directiva | Comportamiento |
|---|---|---|
| Cliente no encontrado | Router + filtro | Registra en Log_Errores, avisa por Slack, el flujo no muere |
| Fallo API de IA (429/503/timeout) | **Error handler Resume** | Log_Errores con el mensaje de error, consulta en Estado: Error |
| Email con cuerpo vacío | Filtro pre-IA | Corta la rama y evita consumir tokens |
| Fallo de envío Gmail | Error handler Break (retry) | Reintenta; si falla, queda en Aprobado para reenvío manual |

## 🙋 Human-in-the-Loop

El Escenario 1 **termina** en la notificación de Slack con dos links (Aprobar/Rechazar) que apuntan a un Custom Webhook con `id_consulta` y `decision`. Nada se envía al cliente sin clic humano — prevención del "efecto metralleta". El Escenario 2 responde **en el mismo hilo de Gmail** usando el Thread ID guardado.

## 🧪 Test de estrés (5 ejecuciones)

1. ✅ Consulta normal aprobada → respuesta enviada en el hilo, Estado: Enviado.
2. ✅ Remitente desconocido → ruta alternativa + Log_Errores.
3. ✅ Cuerpo vacío (camino infeliz) → filtro corta antes de la IA.
4. ✅ Fallo simulado de la API de IA → Resume + registro, el escenario no se rompe.
5. ✅ Rechazo humano → Estado: Rechazado, NO se envía nada al cliente.

## 🔧 Decisiones técnicas (debugging real del desarrollo)

1. **Campo primario de Airtable no admite "Link to record"** → se creó el campo de vínculo aparte y se reubicó el asunto como campo primario (enroque de columnas), manteniendo la relación bidireccional.
2. **El trigger de Gmail en Make separa Folder de Label** → las etiquetas de Gmail no aparecen en Folder; la configuración correcta es `Folder: INBOX` + `Label: Consultas-Clientes`.
3. **Gemini 3.5 Flash truncaba el JSON**: los tokens de razonamiento interno (thinking) se descuentan del límite de salida; con 800 tokens el borrador quedaba cortado y el JSON inválido. Solución: límite de salida realista (4000) + prompt que fuerza JSON en una sola línea sin saltos reales ni comillas dobles internas.

## 📁 Contenido del repo

```
├── README.md
├── Diagrama_Arquitectura_Proyecto_Final.pdf
├── prompt_ia.txt
│   Triage Consultas - Principal.blueprint.json
│   Triage Consultas - Aprobacion HITL.blueprint.json
│   Guia_Configuracion_Airtable.md
│   Clientes.csv · Consultas.csv · Log_Errores.csv
│   Cliente_desconocido.png
│   Cuerpo_vacio.png
│   Escenario_1_Completo.png
│   Escenario_2_Completo.png
│   Flujo_1.png
│   Flujo_Avanzado.png
│   Flujo_Completo.png
│   Flujo_Con_Router.png
│   Kanban_1.png
│   Kanban_Errores.png
│   Mail_De_Aprobacion.png
│   Mail_De_Respuesta.png
│   Proceso_1.png
│   Rechazo_Humano.png
```

## 🎬 Video demo

[https://drive.google.com/file/d/14jilDhzUh74e7CISqq2k1psPKBOG3QG0/view?usp=sharing] — 3 min: trigger → procesamiento en Make → clasificación IA → aprobación humana → respuesta en el hilo. API keys y credenciales ocultas durante toda la grabación.

## 🔒 Seguridad

- API keys almacenadas únicamente en las conexiones de Make (los blueprints exportados no las incluyen).
- Base compartida en modo solo lectura con datos 100% ficticios.
- Ningún dato real de clientes del estudio en este repositorio.

---

*Desarrollado por Jonatan Remolina — estudio de liquidación de sueldos.

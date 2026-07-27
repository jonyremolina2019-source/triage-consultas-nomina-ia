# Guía de configuración — Base Airtable "Triage Consultas Nómina"

Base: **Triage_Consultas_Nomina** · 3 tablas relacionadas · Tiempo estimado: 15-20 min

---

## Paso 1 — Crear la base e importar los CSV

1. En Airtable → **Create a base** → nombrala `Triage_Consultas_Nomina`.
2. Importá cada CSV como tabla nueva: **Add or import → CSV file**. El orden importa: primero `Clientes.csv`, después `Consultas.csv`, por último `Log_Errores.csv`.
3. Borrá la tabla "Table 1" que Airtable crea por defecto.

> Airtable importa todo como texto. Los pasos siguientes convierten cada campo a su tipo correcto — hacelo en orden.

---

## Paso 2 — Tabla `Clientes`

| Campo | Convertir a | Configuración |
|---|---|---|
| Nombre empresa | Single line text | Campo primario (ya lo es). |
| CUIT | Single line text | Sin cambios. |
| Email contacto | **Email** | Clic derecho → Edit field → Email. |
| CCT aplicable | **Single select** | Airtable genera las opciones solo al convertir: FAECYS 130/75, Pasteleros 272/96, Camioneros 40/89, UTEDYC 736/16, SUTERH 589/10, SETIA 501/07. Podés agregar los demás CCT después. |
| Estado | **Single select** | Opciones: `Activo` (verde), `Inactivo` (gris). |

El campo de relación `Consultas` NO lo crees acá — aparece solo en el Paso 3.

---

## Paso 3 — Tabla `Consultas` (la central)

| Campo | Convertir a | Configuración |
|---|---|---|
| Cliente | **Link to another record → Clientes** | Al convertir, Airtable matchea por el nombre exacto y crea automáticamente el campo espejo `Consultas` en la tabla Clientes. Desmarcá "Allow linking to multiple records". |
| Email remitente | **Email** | — |
| Asunto | Single line text | Campo primario. |
| Cuerpo | **Long text** | — |
| Thread ID | Single line text | Lo escribe Make desde el trigger de Gmail. |
| Categoria IA | **Single select** | `Liquidación` (azul), `Documentación` (violeta), `ARCA-LSD` (naranja), `Otro` (gris). Deben coincidir EXACTO con las categorías del prompt del LLM. |
| Urgencia | **Single select** | `Alta` (rojo), `Media` (amarillo), `Baja` (verde). |
| Borrador respuesta | **Long text** | Activá "Enable rich text formatting" = OFF (Make escribe texto plano). |
| Estado | **Single select** | En este orden: `Pendiente` (gris), `Procesado por IA` (azul), `Esperando Aprobación` (amarillo), `Aprobado` (verde claro), `Rechazado` (naranja), `Enviado` (verde), `Error` (rojo). |
| Fecha envio | **Date** | Formato ISO, incluir hora = opcional. |

**Campo extra recomendado**: `ID Consulta` → tipo **Autonumber**. Es el identificador que viaja en el link del webhook de aprobación (`?id_consulta={{ID}}`). Más limpio que usar el Record ID de Airtable en la URL, aunque el Record ID también sirve.

---

## Paso 4 — Tabla `Log_Errores`

| Campo | Convertir a | Configuración |
|---|---|---|
| Modulo | **Single select** | `Filtro cliente`, `Filtro datos faltantes`, `IA_Clasificar_Responder`, `Gmail_Responder_Cliente`, `Airtable`, `Otro`. |
| Mensaje error | **Long text** | Make mapea `{{error.message}}`. |
| Consulta | **Link to another record → Consultas** | Puede quedar vacío (ej.: remitente desconocido no genera consulta). |
| Timestamp | **Crear campo nuevo → Created time** | Automático, no viene en el CSV. |

---

## Paso 5 — Vistas (para el video demo quedan muy bien)

En `Consultas`, creá estas vistas tipo Grid con filtro por Estado:

- **📥 Bandeja pendiente** → Estado is `Pendiente`
- **🤖 Procesadas por IA** → Estado is any of `Procesado por IA`, `Esperando Aprobación`
- **✅ Enviadas** → Estado is `Enviado`
- **⚠️ Errores** → Estado is `Error` (+ vista en Log_Errores ordenada por Timestamp desc)

Y una vista **Kanban agrupada por Estado**: en el video se ve la tarjeta moviéndose de columna en columna a medida que corre el flujo — impacta mucho más que la grilla.

---

## Paso 6 — Compartir en modo lectura (entregable)

1. Botón **Share** (arriba a la derecha de la base) → **Base → Create shared link**.
2. Tipo: **Read-only**. Desactivá "Allow viewers to copy data" si preferís.
3. Ese link es el que va en el README del repo de GitHub.

---

## Checklist de requisitos del proyecto cubiertos

- [x] Campos de estado con el ciclo completo (Pendiente → Procesado por IA → Esperando Aprobación → Aprobado/Rechazado → Enviado | Error)
- [x] Relaciones entre tablas: Consultas↔Clientes y Log_Errores↔Consultas (sin datos aislados)
- [x] Registro persistente de errores (Log_Errores) para el error handling de Make
- [x] Thread ID almacenado para mapear el hilo de Gmail
- [x] Datos de prueba ficticios que cubren los 5 escenarios del test de estrés
- [x] Sincronización de estados gestionada únicamente por el flujo (Airtable = fuente de verdad)

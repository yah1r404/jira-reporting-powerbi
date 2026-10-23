# Jira Reporting Power BI: Mesa de Servicio MDSC

Proyecto de Power BI en formato **PBIP/PBIR** con datos de **Jira Cloud** (`https://secureopsmx.atlassian.net`) a través de la REST API.
Mide el SLA y la eficiencia de la Mesa de Servicio **MDSC** de RAM (Redes y Asesorías del Mayab).

Para abrirlo, abre `Jira_PBI.pbip` con Power BI Desktop.

### Estructura del repositorio

| Ruta | Contenido |
|---|---|
| `Jira_PBI.pbip`, `Jira_PBI.Report/`, `Jira_PBI.SemanticModel/` | Tablero principal de la Mesa de Servicio MDSC (secciones 1 a 6) |
| `IEP_Incident/` | Tablero ejecutivo QuattroCom: `IEP_Incident.pbip`, su reporte, su modelo y `IEP_Incident_Graficas.txt` (títulos y significado de cada gráfica) (sección 7) |
| `Mesa_Servicio/` | **Dashboard nuevo de la Mesa de Servicio** (`Mesa_Servicio.pbip`), reconstruido y optimizado: por ahora *00 - Resumen* y *01 - Time in Status* (sección 8) |
| `assets/` | Logos originales de RAM y QuattroCom (cada reporte guarda su propia copia en `StaticResources`) |
| `CLAUDE.md` | Reglas de trabajo del proyecto (Jira solo lectura, sin secretos, etc.) |

---

## 1. Conexión a Jira

- El acceso es **de solo lectura**. El reporte no crea, modifica ni comenta tickets.
- Autenticación **Básica**:
  - usuario: tu correo de Jira
  - contraseña: tu **API token** de Atlassian (https://id.atlassian.com/manage-profile/security/api-tokens)
- Las credenciales las guarda Power BI en tu equipo. **No están en el proyecto ni en Git.**
- Nivel de privacidad del origen: **Organizational**.
- En **Inicio → Actualizar** se vuelven a bajar los datos.
- **Llamadas a Jira optimizadas:**
  - La búsqueda de tickets es ligera: pide solo los campos que usa el modelo, sin historial y con páginas de hasta 1000 tickets.
  - El historial de **estado y responsable** se pide aparte con `POST /rest/api/3/changelog/bulkfetch`: hasta 1000 tickets por llamada, filtrado a esos dos campos. Solo lo calculan `FactStatusHistory` y `FactAssigneeHistory`.
  - Si ese endpoint fallara, la consulta vuelve sola al método anterior (búsqueda con historial y llamadas por ticket).

## 2. Datos (Power Query)

| Consulta | Qué trae | ¿Se carga al modelo? |
|---|---|---|
| `Jira_MDSC_Issues` | **Todo el histórico** de MDSC **sin tickets en estado Cerrada** (`project = MDSC AND status != Closed ORDER BY created DESC`; incluye Emailed request), con sus campos, su changelog y la hora de su **primer comentario**. Las tablas de abajo salen de aquí. | No |
| `SLA_Tickets` | Un renglón por ticket: flujo, request type, tipo de ticket, **organización**, estado, responsable, fechas y el **SLA calculado con reglas propias** (24/7 y calendario operativo RAM). Los valores de SLA de JSM se conservan ocultos como `JSM …` solo como referencia. | Sí |
| `Calendario SLA` | Tabla calculada del selector *Calendario SLA*: 24/7 (contractual, predeterminado) u Operativo RAM. Sin relaciones. | Sí |
| `SLA Objetivo` | Tabla calculada con las opciones del filtro de SLA del cliente: Según cada ticket (meta en horas de cada ticket), 4, 6, 15 y 18 hrs. No tiene relaciones. | Sí |
| `FactStatusHistory` | Un renglón por cada tramo que un ticket pasó en un estado (**Time in Status**). | Sí |
| `FactTecnicos` | Un renglón por ticket y **técnico de campo** (campo *Técnico(s) asignado(s)*). Excluye cuentas de equipo (*Servicios Equipo…*) y agrupa por nombre (una persona puede tener dos cuentas). | Sí |
| `FactAssigneeHistory` | Un renglón por cada tramo que un ticket estuvo asignado a una persona (**Time in Assignee**). | Sí |

### Campos de Jira usados

| Campo | ID |
|---|---|
| Request Type | `customfield_10010` |
| Primera respuesta: primer comentario de alguien distinto a quien levantó el ticket | `comment`, `reporter` |
| Resolución | `resolutiondate` (Resolved) |
| Tiempo hasta primera respuesta / resolución (SLA JSM, solo referencia, oculto) | `customfield_10936` / `customfield_10937` |
| Técnico(s) asignado(s) | `customfield_11042` |
| Organizations (cliente) | `customfield_10002` |
| Tipo de ticket (radio buttons) | se busca por nombre al actualizar |

### Flujos de la mesa

| Flujo | Tipos de ticket | Estados principales |
|---|---|---|
| **classic default workflow** (en el tablero, tipo de servicio = "Proyecto Internet en Parques (IEP)") | IEP INCIDENT | Nueva solicitud, N1, N2 RAM, N2 QCOM, Planta externa, Esperando por el cliente, Escalado, Resuelta, Reabierta, Cerrada |
| **Mesa de Servicio - Clientes** (tipo de servicio = "Mesa de Servicio") | Emailed request, Tarea y el resto | Nueva solicitud, N1, N2 RAM, N3 RAM, Esperando por el cliente, Escalado, Esperando al proveedor, Resuelta, Reabierta, Cerrada |

Los nombres en inglés del historial se normalizan al español: Waiting for customer → Esperando por el cliente, Escalated → Escalado, Resolved → Resuelta, Reopened → Reabierta, Closed → Cerrada. `FactStatusHistory[Estado Jira]` conserva el nombre original.

Los dos flujos comparten "Esperando por el cliente", "Escalado" y "Reabierta". El orden de los estados para gráficos y tablas se define en la lista `OrdenEstados` de la consulta `FactStatusHistory`.

## 3. Modelo

```
DimFecha ──1:*──> SLA_Tickets             (Created Date)
SLA_Tickets ──1:*──> FactStatusHistory    (Key)
SLA_Tickets ──1:*──> FactAssigneeHistory  (Key)
SLA_Tickets ──1:*──> FactTecnicos         (Key)
```

Todas las relaciones son de uno a muchos y en una sola dirección. `DimFecha` es un calendario calculado que cubre las fechas de `SLA_Tickets`.

### Medidas (tabla `_Medidas`)

| Carpeta | Medidas |
|---|---|
| SLA | Tickets, Tickets Resueltos, Tickets Abiertos, Última Actualización |
| SLA \ Resolución | SLA Resolución Cumplidos / Incumplidos / En Riesgo, **% SLA Resolución**, Prom Horas SLA Resolución |
| SLA \ Primera Respuesta | SLA Respuesta Cumplidos / Incumplidos, **% SLA Primera Respuesta**, Prom Horas SLA Respuesta |
| SLA \ Calendario | Prom / Mediana / P90 Horas Resolución Calendario |
| SLA \ Semáforo | Meta % SLA (95%), Umbral Alerta % SLA (85%), colores de semáforo |
| Time in Status | Horas en Estado, Tickets en Estado, Prom / Mediana / P90 Horas en Estado |
| SLA Seleccionado | Meta SLA Seleccionada, Meta SLA Aplicada, Cumplen / Incumplen / Evaluados SLA Seleccionado, **% SLA Seleccionado**, Horas Ticket Incumplido, Meta Ticket Incumplido, Horas Excedidas SLA, Horas en Estado Incumplidos |
| Filtros | Filtros Activos (texto de los filtros aplicados) |
| Técnicos de Campo | Tickets con Técnico (únicos), Participaciones Técnicos, Técnicos Involucrados, Tickets por Técnico, Técnicos por Ticket, Abiertos / En Proceso / Resueltos / Incumplidos, % Participación Técnico, % SLA Resolución Técnico, Prom Horas SLA Técnico, Mediana Horas Resolución Técnico, Técnicos en el Ticket |
| Time in Assignee | Horas Asignado, Tickets Atendidos, Prom Horas Asignado por Ticket, Mediana Horas Asignado, Personas por Ticket, Prom Horas Sin Asignar, Total Reasignaciones, % SLA Resolución Tickets Atendidos |
| Línea de Tiempo | Gantt Responsable, Gantt Estado y sus colores (tabla auxiliar `Línea de Tiempo`, horas 0 a 48+) |

### Reglas de cálculo

- **SLA calculado en el dashboard (no se usan las metas ni los estados de SLA de Jira):**

  | Flujo | Tipo de solicitud | Primera respuesta | Resolución |
  |---|---|---|---|
  | classic default workflow | IEP INCIDENT | 30 min | 15 h |
  | Mesa de Servicio - Clientes | Emailed request, Tarea y el resto | 4 h | 24 h |

- **Primera respuesta** = de `created` al **primer comentario** de alguien distinto a quien levantó el ticket (de cualquier visibilidad). Sin comentario: si el ticket sigue abierto, su tiempo corre (*En curso* o *Incumplido* si ya pasó la meta); si ya se resolvió, queda **Sin dato** y no entra al %.
- **Resolución** = de `created` a **Resolved**. Si Resolved está vacío y el ticket está en Resuelta/Cerrada, se usa su **última entrada a Resuelta** del historial (columna *Fuente Fecha Resolución*). Si sigue abierto, el tiempo es **acumulado** hasta el último refresh (no es una resolución).
- **Pausas** del SLA de resolución: Esperando por el cliente, Esperando al proveedor, Resuelta y Cerrada, a partir de los tramos del historial (con reaperturas, el tiempo en Resuelta antes de reabrir no cuenta). Cada tramo se recorta al fin del conteo, sin traslapes.
- **Calendarios:** *24/7 (contractual)* = tiempo real menos pausas. *Operativo RAM* = solo horas dentro de **L-V 7:00-23:00 y sáb-dom 8:00-18:00**, menos pausas. El selector **Calendario SLA** (sincronizado; 24/7 si no se elige) cambia todas las medidas de SLA; la página *09 - SLA por Calendario* muestra los dos lado a lado.
- **Estado del SLA**: *Cumplido*, *Incumplido* (incluye abiertos que ya pasaron la meta), *En riesgo* (abierto con 75% o más de la meta), *Pausado* (abierto en un estado de pausa), *En curso* y *Sin dato*.
- **% SLA** = Cumplidos / (Cumplidos + Incumplidos). En curso, en riesgo, pausados y sin dato no cuentan y se reportan aparte.
- **Situación del ticket**: Resuelto, Resuelto (tuvo reapertura), Reabierto (abierto) o Abierto.
- **Estados de flujos anteriores**: su tiempo se mide en el estado actual que los representa:
  - Open, Abierto(s), Nuevo, To Do, Waiting for support o cualquier estado inicial (salvo Reabierta) → **Nueva solicitud**
  - In progress, En curso, Trabajo en curso, Pendiente, Pending, N1 QCOM, N1 RAM → **N1**
  - Escalado a N2, N2 → **N2 RAM** · Escalado a N3, N3 → **N3 RAM**
  - Esperando al solicitante, Waiting for customer → **Esperando por el cliente** · Completado, Resolved → **Resuelta**
  Los que no tienen equivalente quedan con `Vigente` = FALSE: cuentan para el SLA pero no aparecen en *Time in Status*. El tiempo en Resuelta y Cerrada no cuenta ni para el SLA ni para Time in Status.
- **Horario de cálculo** de Time in Status y Time in Assignee: **operativo RAM** (L-V 7:00-23:00, sáb-dom 8:00-18:00; predeterminado) o **24/7**. Se define en la función `HorasLaborales` (columna `Horas Hábiles` de cada historial).
- **Time in Status** está en horas y no incluye estados finales (Resuelta, Cerrada). Si un ticket sigue en un estado, su tramo cuenta hasta la hora del último refresh.
- **Técnicos de campo**: un ticket con varios técnicos cuenta **una vez por técnico** (participaciones) y **una sola vez** en *Tickets con Técnico* (únicos). El SLA y los tiempos se **atribuyen por participación**, no por responsabilidad (el responsable es el Assignee). Abiertos = categoría *Por hacer*; en proceso = sin resolver y ya en atención; resueltos = con fecha de resolución.
- **Time in Assignee** está en horas según el horario de cálculo, desde la creación hasta la resolución, y **solo cuenta tickets resueltos**. El tiempo sin responsable no se atribuye a nadie (se ve aparte en *Prom Horas Sin Asignar*).
- **Reasignaciones**: cambios de responsable, sin contar la primera asignación.
- **Vista Ejecutiva consistente con el resto:** con *SLA objetivo* = "Según cada ticket" (predeterminado), cumple/no cumple es el **mismo estado del SLA calculado** que usan las demás páginas, así que los números coinciden. Solo al elegir una meta del cliente (4, 6, 15 o 18 h) se recalcula contra esas horas.
- **SLA objetivo (filtro)**: cada ticket se compara contra las horas elegidas en el filtro *SLA objetivo*; con "Según cada ticket" se usa la meta en horas de cada ticket (ej. 15 h o 24 h), y la tarjeta muestra esas horas. Las horas comparadas (`Horas SLA Base` / `Horas SLA Base RAM`) son el tiempo de SLA de resolución calculado, sin pausas, en el calendario elegido. Un ticket abierto que ya pasó la meta cuenta como incumplido.
- **Fechas** en hora de Mérida (UTC-6).
- El **semáforo** (verde ≥ 95%, ámbar ≥ 85%, rojo debajo) usa metas provisionales. Se ajustan en las medidas `Meta % SLA` y `Umbral Alerta % SLA`.

## 4. Páginas del reporte

| Página | Datos | Contenido |
|---|---|---|
| **00 - Resumen** | Mesa MDSC | 1920 × 1080. Pirámides de volumen, SLA de primera respuesta y de resolución (con metas, cumplen / no cumplen y tiempos), cumplimiento por estado, mensual, creados vs. resueltos, por hora y por día |
| **01 - SLA** | Mesa MDSC | 1920 × 1080. Total, % SLA 1ª respuesta y resolución (con metas y cumplen / no cumplen), tiempos promedio; cumplimiento por mes y por día de la semana; % por tipo de solicitud y por tipo de servicio; tabla por tipo de solicitud |
| **02 - Time in Status** | Mesa MDSC | 1920 × 1080. Tickets analizados, tiempo promedio en estados, estado con más tiempo, % en espera, tiempo sin atender; horas por estado y tipo de servicio, tendencia por semana y mes, tablas por tipo de servicio y por request type. Solo estados vigentes; selector de horario |
| **04 - Detalle SLA** | Mesa MDSC | Tabla ticket por ticket con estado de SLA y link a Jira |
| **06 - Time in Assignee** | Mesa MDSC | Horas promedio asignado por ticket resuelto y tickets atendidos por persona. Filtro propio: responsable |
| **07 - Técnicos de Campo** | Mesa MDSC | Participación individual de los técnicos de campo según *Técnicos asignados* (complementa a Time in Assignee, que mide por Assignee): KPIs, tickets por técnico, abiertos/en proceso/resueltos, tipo de trabajo por request type, tendencia mensual, SLA y tiempos de sus tickets y detalle de tickets por técnico. Filtros propios: técnico de campo y estado del ticket. Página más alta (1800 px, con scroll) |
| **08 - Línea de Tiempo** | Mesa MDSC | Diagrama tipo Gantt por ticket, hora por hora desde la creación (0h a 48h+): quién lo tuvo y en qué estado estuvo. Filtro propio: ticket |

Todas las páginas comparten la misma estructura: barra lateral con el logo y el menú (página activa en amarillo), encabezado con título y descripción, barra de filtros y una línea con los **filtros activos**.

### Filtros

| Tipo | Filtros | Dónde aplican |
|---|---|---|
| **Globales sincronizados** | Periodo (fecha de creación), Cliente (organización), Tipo de Servicio (flujo), Tipo de Solicitud (request type) | Todas las páginas. Al cambiar de página se conservan. |
| **Propios de la página** | Responsable (06), Técnico de campo (07), Ticket (08) | Solo en su página; no modifican los globales |

- Todas las fechas son de **creación del ticket**, en hora de Mérida (UTC-6). El rango incluye el último día.

## 5. Diseño

- Tema corporativo **RAM** en `Jira_PBI.Report/StaticResources/RegisteredResources/RAM_Tema.json`. Es la fuente única de colores, tipografía, tarjetas, tablas y gráficos de todo el reporte.
- Colores tomados del logo: azul `#06519A` (principal), amarillo `#F1D72C` (acento), azul oscuro `#0A2E5C` (títulos) y grises neutros.
- **Tipografía única: Segoe UI** (regular, Semibold y Bold). Power BI no permite fuentes externas como Inter.
- **Botones**, con contraste WCAG AA en sus estados normal, hover y presionado:
  - Navegación: azul con texto blanco; activo en amarillo con texto azul oscuro.
  - Primario: azul con texto blanco.
  - Secundario: blanco con borde y texto azul oscuro.
  - Acento: amarillo con texto azul oscuro.
- **Colores con significado** (iguales en todas las páginas):

  | Significado | Color | Dónde |
  |---|---|---|
  | Neutro / cumplido | Azul `#06519A` | SLA cumplido, % SLA en meta |
  | En curso | Azul claro `#6A9BD8` | SLA en curso |
  | En riesgo / atención | Amarillo `#F1D72C` (texto ámbar `#8A6100`) | SLA en riesgo, % SLA bajo el umbral |
  | No cumple / urgente | Rojo `#C8382E` | SLA incumplido, % SLA bajo el umbral de alerta, tarjetas e importes de incumplimiento |
  | Sin dato | Gris `#A9B4C2` | Sin SLA |

  Las reglas viven en las medidas de la carpeta *Colores* (`Color Estado SLA Resolución`, `Color Estado SLA Respuesta`, `Color Alerta`) y *SLA \ Semáforo*.
- Logo en `Jira_PBI.Report/StaticResources/RegisteredResources/redesmayab_logo.jpg`.

## 6. Pendientes

- Confirmar las metas reales de % SLA para el semáforo.
- Validar los números contra Jira después del primer refresh completo.

## 7. Tablero ejecutivo QuattroCom (`IEP_Incident/IEP_Incident.pbip`)

Proyecto aparte en la carpeta `IEP_Incident/`. Usa las mismas credenciales de Jira y el **tema QuattroCom** (rojo, negro y blanco, con azul marino y verde en gráficas). Tiene 2 páginas de 1920 × 1080, ajustadas a la página. `IEP_Incident/IEP_Incident_Graficas.txt` explica cada tarjeta, gráfica y columna.

**Identidad:** el panel lateral muestra los logos de **RAM** y **QuattroCom** (`redesmayab_logo.jpg` y `quattrocom_logo.png` en `IEP_Incident.Report/StaticResources/RegisteredResources/`).

**Alcance fijo:** `project = MDSC AND type = "IEP INCIDENT" AND "tipo de ticket[radio buttons]" = "Incidente Qcom"` (en la consulta: `cf[10894] = "Incidente Qcom"`). Incluye todo el historial y también los tickets en Cerrada.

**Filtros sincronizados en las 2 páginas:** Fecha (de creación) y Tipo de falla (campo *Causa* de Jira).

**Página 1 · Resumen ejecutivo:** total, resueltos y pendientes (con % del total); tiempo promedio a primera respuesta y a resolución; % de cumplimiento por SLA con cumplidos e incumplidos (con % del total); cumplimiento de los SLA por estado (% y cantidad); cumplimiento mensual; creados vs. resueltos por mes; por hora y por día de la semana.

**Página 2 · Análisis y detalle:** incidentes por tipo de falla; tarjeta de Planta externa (% de tickets que pasaron por ahí); **Detalle de tickets** con SLA target, tiempo SLA (rojo si venció, verde si cumplió), una columna de tiempo por estado (Nueva solicitud, N1, N2, Esperando por el cliente, Planta externa, Otros estados), situación, creación, resuelto, vida total y enlace a Jira.

**Definiciones:**

| Indicador | Cálculo |
|---|---|
| Fecha de resolución | Campo **Resolved** de Jira; si un ticket en Resuelta no lo tiene, la hora en que entró a Resuelta. Todas las medidas de resolución la usan. |
| Total de incidentes | Creados en el rango de fechas |
| Resueltos | Con fecha de resolución dentro del rango |
| Pendientes | Creados en el rango que hoy no tienen fecha de resolución |
| % SLA | Cumplidos / (cumplidos + incumplidos), con **las mismas reglas para todos los tickets**: primera respuesta en **30 min** y resolución en **15 h**, calendario de **24 h**. El tiempo de resolución va de la creación a la fecha de resolución (o al último refresh si sigue abierto) y descuenta el tiempo en pausa (Esperando por el cliente, Planta externa y Cerrada). La primera respuesta usa el dato de Jira; si no existe, el primer cambio de estado. Los valores originales de Jira se conservan ocultos como columnas `JSM …`. |
| Tiempo por estado (tabla) | Tiempo en cada estado hasta la fecha de resolución, todas las visitas sumadas. N1 incluye N1 QCOM y N1 RAM y N2 incluye N2 RAM y N2 QCOM. Los estados que ya no existen en el flujo se conservan con su nombre del historial. |
| Vida total | Tiempo natural de la creación a la fecha de resolución, sin descontar pausas |

**Pendiente de confirmar:** las fechas de diagnóstico y de atención del servicio todavía no están en la tabla. Falta identificar en Jira qué campos son y qué tan llenos están.

## 8. Dashboard de Mesa de Servicio optimizado (`Mesa_Servicio/Mesa_Servicio.pbip`)

Reconstrucción de `Jira_PBI` con solo lo necesario, para que actualice más rápido y sea fácil de ampliar. Mismas reglas de SLA que la sección 3 (24/7 y operativo RAM, pausas, primer comentario, Resolved con respaldo en el historial).

**Páginas (1920 × 1080):** *00 - Resumen* (pirámides de volumen y SLA, cumplimiento por estado, mensual, creados vs. resueltos, por hora y por día) y *01 - Time in Status* (horas promedio por estado y flujo, tabla por flujo/estado y por request type; selector de horario laboral/24/7).

**Modelo (8 tablas, 54 medidas):**

```
DimFecha ──1:*──> Tickets ──1:*──> Historial
'Calendario SLA', 'Horario Cálculo', 'SLA Comparación', 'Estado SLA'  (selectores, sin relaciones)
```

| Elemento | Qué es | Optimización frente a Jira_PBI |
|---|---|---|
| `Jira_Fuente` (no se carga) | Única consulta a Jira: **una búsqueda paginada** con 11 campos, los comentarios y el historial (`expand=changelog`); solo los tickets con más de 100 cambios piden su historial aparte | Una búsqueda en vez de tres; sin `/field` ni campos de JSM/tipo de ticket. (El historial en bloque por POST no se usa: Power BI no permite POST con usuario y token.) |
| `Tickets` | Un renglón por ticket: atributos, **Assignee** (y su ID), **Técnicos Asignados** (texto y ruta `a|b`), primer comentario, metas y SLA calculado | Metas y horas RAM a la primera respuesta se calculan en Power Query, no en DAX |
| `Historial` | Un renglón por tramo, con columna `Campo` (*Estado*; *Responsable* cuando se active) | **Una sola tabla** para estado y, después, responsable (en Jira_PBI eran dos tablas con su propia descarga del historial) |

**Para crecer sin rehacer la base:**
- *Time in Assignee*: en `Historial` cambiar `CamposCargados = {"status"}` por `{"status", "assignee"}` (el historial ya viene con ambos campos); los tramos de responsable entran a `Historial` con `Campo = "Responsable"` y `Valor` = persona.
- *Técnicos asignados*: la columna `Técnicos Ruta` ya trae los técnicos separados por `|` para generar una tabla puente (un renglón por ticket y técnico) sin otra llamada a Jira. Son una dimensión distinta del assignee (el assignee puede ser una cuenta de equipo como *Servicios Equipo*).


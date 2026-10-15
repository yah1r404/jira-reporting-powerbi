# Jira Reporting Power BI: Mesa de Servicio MDSC

Proyecto de Power BI en formato **PBIP/PBIR** con datos de **Jira Cloud** (`https://secureopsmx.atlassian.net`) a través de la REST API.
Mide el SLA y la eficiencia de la Mesa de Servicio **MDSC** de RAM (Redes y Asesorías del Mayab).

Para abrirlo, abre `Jira_PBI.pbip` con Power BI Desktop.

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
| `Jira_MDSC_Issues` | Tickets de **MDSC creados en los últimos 6 meses**, **sin Emailed request y sin tickets en estado Cerrada** (`project = MDSC AND issuetype != "Emailed request" AND status != Closed AND created >= startOfMonth(-5)`), con sus campos y su changelog completo. Las tres tablas de abajo salen de aquí. | No |
| `SLA_Tickets` | Un renglón por ticket: flujo, request type, tipo de ticket, **organización**, estado, responsable, fechas y los **SLA de Jira Service Management**. | Sí |
| `SLA Objetivo` | Tabla calculada con las opciones del filtro de SLA del cliente: Según cada ticket (meta en horas de cada ticket), 4, 6, 15 y 18 hrs. No tiene relaciones. | Sí |
| `FactStatusHistory` | Un renglón por cada tramo que un ticket pasó en un estado (**Time in Status**). | Sí |
| `FactTecnicos` | Un renglón por ticket y **técnico de campo** (campo *Técnico(s) asignado(s)*). Excluye cuentas de equipo (*Servicios Equipo…*) y agrupa por nombre (una persona puede tener dos cuentas). | Sí |
| `FactAssigneeHistory` | Un renglón por cada tramo que un ticket estuvo asignado a una persona (**Time in Assignee**). | Sí |

### Campos de Jira usados

| Campo | ID |
|---|---|
| Request Type | `customfield_10010` |
| Tiempo hasta primera respuesta (SLA JSM, meta 30 min) | `customfield_10936` |
| Tiempo hasta resolución (SLA JSM, meta 15 h / 24 h) | `customfield_10937` |
| Técnico(s) asignado(s) | `customfield_11042` |
| Organizations (cliente) | `customfield_10002` |
| Tipo de ticket (radio buttons) | se busca por nombre al actualizar |

### Flujos de la mesa

| Flujo | Tipos de ticket | Estados principales |
|---|---|---|
| **IEP Incident** | IEP INCIDENT | Nueva solicitud → N1 → N2 RAM / N2 QCOM → Escalado QCOM → Resuelta → Cerrada |
| **Mesa de Servicio - Clientes** | Task y los demás tipos (los Emailed request se excluyen) | Nueva solicitud → En curso → Escalado a N2 / N3 → Esperando al proveedor → Resuelta → Cerrada |

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

- **% SLA** = Cumplidos / (Cumplidos + Incumplidos). Los tickets "En curso" y "Sin SLA" no cuentan.
- **Estado del SLA** (lo calcula Jira Service Management, respetando pausas y calendario):
  - *Cumplido*
  - *Incumplido*: incluye tickets abiertos que ya vencieron.
  - *En riesgo*: abierto con 25% o menos del tiempo restante.
  - *En curso*
  - *Sin SLA*
- **Estados vigentes**: solo se muestran los estados que tiene hoy el flujo de cada ticket en Jira. El tiempo en estados de flujos anteriores se descarta. Los estados se identifican por su ID y se muestran con su nombre actual.
- **Horario de cálculo** (selector en Time in Status, Time in Assignee y Filtros avanzados; sincronizado): **horario laboral** (lunes a viernes 6:00-23:00 y sábado 9:00-19:00; domingo no cuenta), que es el predeterminado, o **24/7** (horas de reloj). Se define en la función `HorasLaborales` y se guarda en la columna `Horas Hábiles` de cada historial. No cambia el % de SLA, que usa el calendario del SLA en Jira, ni la línea de tiempo, que va hora por hora.
- **Time in Status** está en horas y no incluye estados finales (Resuelta, Cerrada). Si un ticket sigue en un estado, su tramo cuenta hasta la hora del último refresh.
- **Técnicos de campo**: un ticket con varios técnicos cuenta **una vez por técnico** (participaciones) y **una sola vez** en *Tickets con Técnico* (únicos). El SLA y los tiempos se **atribuyen por participación**, no por responsabilidad (el responsable es el Assignee). Abiertos = categoría *Por hacer*; en proceso = sin resolver y ya en atención; resueltos = con fecha de resolución.
- **Time in Assignee** está en horas según el horario de cálculo, desde la creación hasta la resolución, y **solo cuenta tickets resueltos**. El tiempo sin responsable no se atribuye a nadie (se ve aparte en *Prom Horas Sin Asignar*).
- **Reasignaciones**: cambios de responsable, sin contar la primera asignación.
- **Vista Ejecutiva consistente con el resto:** con *SLA objetivo* = "Según cada ticket" (predeterminado), cumple/no cumple es el **mismo estado del SLA de Jira** que usan las demás páginas, así que los números coinciden. Solo al elegir una meta del cliente (4, 6, 15 o 18 h) se recalcula contra esas horas.
- **SLA objetivo (filtro)**: cada ticket se compara contra las horas elegidas en el filtro *SLA objetivo*; con "Según cada ticket" se usa la meta en horas de cada ticket (ej. 15 h o 24 h), y la tarjeta muestra esas horas. Las horas comparadas (`Horas SLA Base`) son el tiempo de SLA de resolución de JSM, que respeta pausas y horario; si el ticket no tiene SLA en JSM, se usan las horas de reloj. Un ticket abierto que ya pasó la meta cuenta como incumplido.
- **Fechas** en hora de Mérida (UTC-6).
- El **semáforo** (verde ≥ 95%, ámbar ≥ 85%, rojo debajo) usa metas provisionales. Se ajustan en las medidas `Meta % SLA` y `Umbral Alerta % SLA`.

## 4. Páginas del reporte

| Página | Datos | Contenido |
|---|---|---|
| **01 - Vista Ejecutiva** | Mesa MDSC | Selector de **SLA objetivo del cliente** (solo esta página). % de cumplimiento con semáforo, tendencia mensual, estados donde se va más tiempo por flujo, **lista de tickets incumplidos con su clave de Jira** y horas por estado de cada incumplido |
| **02 - SLA Resumen** | Mesa MDSC | KPIs de SLA con semáforo; el subtítulo muestra las horas de las metas (primera respuesta y resolución); tendencia mensual, % SLA por request type, tickets por flujo y estado de SLA |
| **03 - SLA por Tipo de Ticket** | Mesa MDSC | % SLA por tipo de servicio y tipo de solicitud; detalle de cumplimiento y tiempos |
| **04 - Detalle SLA** | Mesa MDSC | Tabla ticket por ticket con estado de SLA y link a Jira |
| **05 - Time in Status** | Mesa MDSC | Horas promedio por estado vigente, comparando los dos flujos, con mediana y P90 |
| **06 - Time in Assignee** | Mesa MDSC | Horas promedio asignado por ticket resuelto y tickets atendidos por persona. Filtro propio: responsable |
| **07 - Técnicos de Campo** | Mesa MDSC | Participación individual de los técnicos de campo según *Técnicos asignados* (complementa a Time in Assignee, que mide por Assignee): KPIs, tickets por técnico, abiertos/en proceso/resueltos, tipo de trabajo por request type, tendencia mensual, SLA y tiempos de sus tickets y detalle de tickets por técnico. Filtros propios: técnico de campo y estado del ticket. Página más alta (1800 px, con scroll) |
| **08 - Línea de Tiempo** | Mesa MDSC | Diagrama tipo Gantt por ticket, hora por hora desde la creación (0h a 48h+): quién lo tuvo y en qué estado estuvo. Filtro propio: ticket |
| Filtros avanzados *(oculta en el menú)* | Mesa MDSC | Panel de filtros en modo borrador: se abre con el botón «Filtros avanzados» |

Todas las páginas comparten la misma estructura: barra lateral con el logo y el menú (página activa en amarillo), encabezado con título y descripción, barra de filtros y una línea con los **filtros activos**.

### Filtros

| Tipo | Filtros | Dónde aplican |
|---|---|---|
| **Globales sincronizados** | Periodo (fecha de creación), Cliente (organización), Tipo de Servicio (flujo), Tipo de Solicitud (request type) | Todas las páginas. Al cambiar de página se conservan. |
| **Adicionales** (en «Filtros avanzados») | Tipo de ticket, Estado del SLA de resolución, Estado actual, Responsable actual | Todas las páginas, sincronizados de forma oculta |
| **Propios de la página** | Responsable (06), Técnico de campo (07), Ticket (08). El **SLA objetivo** está en 01 y 08 y se sincroniza entre ambas; el **Estado del ticket** de la 07 es el mismo filtro de Filtros avanzados | Solo en su página; no modifican los globales |

- **Limpiar filtros** quita todos los filtros de la página y, por la sincronía, los globales en todas las páginas. Para quitar uno solo, usa el borrador del encabezado de ese filtro.
- **Filtros avanzados**: los cambios no se aplican hasta pulsar **Aplicar filtros**. **Cancelar** regresa a la página anterior sin aplicarlos.
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


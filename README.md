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
- En **Inicio → Actualizar** se vuelven a bajar los datos. Como se lee el historial (changelog) de cada ticket, la actualización tarda varios minutos.

## 2. Datos (Power Query)

| Consulta | Qué trae | ¿Se carga al modelo? |
|---|---|---|
| `Jira_Tickets` | Tickets del **filtro 10741** de Jira (incidentes IEP del mes). Es la consulta original del proyecto. | Sí |
| `Jira_MDSC_Issues` | Todos los tickets de **MDSC creados en los últimos 6 meses** (`project = MDSC AND created >= startOfMonth(-5)`), con sus campos y su changelog completo. Las tres tablas de abajo salen de aquí. | No |
| `SLA_Tickets` | Un renglón por ticket: flujo, request type, tipo de ticket, prioridad, estado, responsable, fechas y los **SLA de Jira Service Management**. | Sí |
| `FactStatusHistory` | Un renglón por cada tramo que un ticket pasó en un estado (**Time in Status**). | Sí |
| `FactAssigneeHistory` | Un renglón por cada tramo que un ticket estuvo asignado a una persona (**Time in Assignee**). | Sí |

### Campos de Jira usados

| Campo | ID |
|---|---|
| Request Type | `customfield_10010` |
| Tiempo hasta primera respuesta (SLA JSM, meta 30 min) | `customfield_10936` |
| Tiempo hasta resolución (SLA JSM, meta 15 h / 24 h) | `customfield_10937` |
| Técnico(s) asignado(s) | `customfield_11042` |
| Tipo de ticket (radio buttons) | se busca por nombre al actualizar |

### Flujos de la mesa

| Flujo | Tipos de ticket | Estados principales |
|---|---|---|
| **IEP Incident** | IEP INCIDENT | Nueva solicitud → N1 → N2 RAM / N2 QCOM → Escalado QCOM → Resuelta → Cerrada |
| **Mesa de Servicio - Clientes** | Task, Emailed request y los demás tipos | Nueva solicitud → En curso → Escalado a N2 / N3 → Esperando al proveedor → Resuelta → Cerrada |

Los dos flujos comparten "Esperando por el cliente", "Escalado" y "Reabierta". El orden de los estados para gráficos y tablas se define en la lista `OrdenEstados` de la consulta `FactStatusHistory`.

## 3. Modelo

```
DimFecha ──1:*──> Jira_Tickets            (Created Date)
DimFecha ──1:*──> SLA_Tickets             (Created Date)
SLA_Tickets ──1:*──> FactStatusHistory    (Key)
SLA_Tickets ──1:*──> FactAssigneeHistory  (Key)
```

Todas las relaciones son de uno a muchos y en una sola dirección. `DimFecha` es un calendario calculado que cubre las fechas de `Jira_Tickets` y de `SLA_Tickets`.

### Medidas (tabla `_Medidas`)

| Carpeta | Medidas |
|---|---|
| *(raíz)* | Medidas originales sobre `Jira_Tickets`: Total Incidentes, promedios, mediana y percentiles de resolución, rangos de horas |
| SLA | Tickets, Tickets Resueltos, Tickets Abiertos, Última Actualización |
| SLA \ Resolución | SLA Resolución Cumplidos / Incumplidos / En Riesgo, **% SLA Resolución**, Prom Horas SLA Resolución |
| SLA \ Primera Respuesta | SLA Respuesta Cumplidos / Incumplidos, **% SLA Primera Respuesta**, Prom Horas SLA Respuesta |
| SLA \ Calendario | Prom / Mediana / P90 Horas Resolución Calendario |
| SLA \ Semáforo | Meta % SLA (95%), Umbral Alerta % SLA (85%), colores de semáforo |
| Time in Status | Horas en Estado, Tickets en Estado, Prom / Mediana / P90 Horas en Estado |
| Time in Assignee | Horas Asignado, Tickets Atendidos, Prom Horas Asignado por Ticket, Mediana Horas Asignado, Total Reasignaciones, % SLA Resolución Tickets Atendidos |

### Reglas de cálculo

- **% SLA** = Cumplidos / (Cumplidos + Incumplidos). Los tickets "En curso" y "Sin SLA" no cuentan.
- **Estado del SLA** (lo calcula Jira Service Management, respetando pausas y calendario):
  - *Cumplido*
  - *Incumplido*: incluye tickets abiertos que ya vencieron.
  - *En riesgo*: abierto con 25% o menos del tiempo restante.
  - *En curso*
  - *Sin SLA*
- **Time in Status** está en horas de reloj y no incluye estados finales (Resuelta, Cerrada). Si un ticket sigue en un estado, su tramo cuenta hasta la hora del último refresh.
- **Time in Assignee** está en horas de reloj y se corta en la fecha de resolución. El tiempo sin responsable aparece como "Sin asignar".
- **Reasignaciones**: cambios de responsable, sin contar la primera asignación.
- **Fechas** en hora de Mérida (UTC-6).
- El **semáforo** (verde ≥ 95%, ámbar ≥ 85%, rojo debajo) usa metas provisionales. Se ajustan en las medidas `Meta % SLA` y `Umbral Alerta % SLA`.

## 4. Páginas del reporte

| Página | Contenido |
|---|---|
| 01 - Resumen Ejecutivo | Página original sobre el filtro 10741 (incidentes IEP) |
| 02 - Tiempos de Resolución | Página original: tiempos calendario de resolución |
| 03 - Responsables | Página original: análisis por responsable |
| 04 - Detalle de Incidentes | Página original: tabla de incidentes |
| **05 - SLA Resumen** | KPIs de SLA con semáforo, tendencia mensual, % SLA por request type, tickets por flujo y estado de SLA |
| **06 - SLA por Tipo de Ticket** | % SLA por flujo, request type y prioridad; detalle de cumplimiento y tiempos |
| **07 - Time in Status** | Horas promedio por estado comparando los dos flujos, con mediana y P90 |
| **08 - Time in Assignee** | Horas promedio asignado y tickets atendidos por persona, % SLA de sus tickets |
| **09 - Detalle SLA** | Tabla ticket por ticket con estado de SLA y link a Jira |

Las páginas 05 a 09 tienen barra lateral de navegación, encabezado con la hora de la última actualización y filtros por mes, flujo, request type, prioridad y tipo de ticket (o responsable).

## 5. Diseño

- Tema corporativo **RAM** en `Jira_PBI.Report/StaticResources/RegisteredResources/RAM_Tema.json`. Se aplica a todo el reporte.
- Colores tomados del logo: azul `#06519A` (principal), amarillo `#F1D72C` (acento), azul oscuro `#0A2E5C` (títulos) y grises neutros.
- Logo en `Jira_PBI.Report/StaticResources/RegisteredResources/redesmayab_logo.jpg`.

## 6. Pendientes

- Confirmar las metas reales de % SLA para el semáforo.
- Validar los números contra Jira después del primer refresh completo.
- Decidir si las páginas 01 a 04 también llevan la barra lateral y el nuevo diseño.

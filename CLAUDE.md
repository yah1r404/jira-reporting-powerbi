# Jira Reporting Power BI

Este proyecto contiene un modelo Power BI alimentado
desde Jira Cloud mediante Jira REST API.

## Jira

- Jira es la fuente oficial de datos.
- El acceso a Jira debe ser de solo lectura por defecto.
- No modificar tickets sin autorización explícita.
- No crear tickets sin autorización explícita.
- No cambiar estados.
- No cambiar assignees.
- No agregar comentarios sin autorización.
- No ejecutar operaciones masivas.

## Seguridad

- Nunca almacenar API tokens.
- Nunca almacenar contraseñas.
- Nunca subir secretos a Git.
- No modificar conexiones sin autorización.

## Power Query

- Mantener consultas reproducibles.
- Evitar campos innecesarios.
- Mantener tipos de datos correctos.
- Separar extracción de transformación cuando sea necesario.

## Modelado

- Preferir modelo estrella cuando agregue valor.
- Evitar relaciones many-to-many innecesarias.
- Evitar bidireccionales sin justificación.
- Mantener medidas DAX separadas de columnas calculadas cuando corresponda.

## Power BI

- No eliminar tablas o medidas sin autorización.
- Explicar cambios antes de modificar el modelo.
- Mantener compatibilidad con PBIP/PBIR.

## Git

- Revisar cambios antes de commit.
- Mantener cambios pequeños y trazables.
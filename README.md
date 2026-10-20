# Checkpoint 4 – Sincronización del Cerebro Agéntico con Ecosistemas de Negocio

**Alumno:** Santiago Pacheco · **Módulo 4** del curso de AI Automation avanzado (n8n)
**Proyecto integrador:** extiende el workflow del Módulo 3 (Manager/Workers + memoria Airtable). No se rehizo desde cero: se agregó una segunda rama de entrada (Gmail) conectada a HubSpot, Gmail (borrador) y Slack.

## Contenido del repositorio

| Archivo | Descripción |
|---|---|
| `checkpoint4_santiago_pacheco.json` | Workflow principal (Manager + rama M4). **Entregable.** |
| `Worker1_Consulta_Clientes_Santiago_Pacheco.json` | Sub-workflow Worker 1 (consulta de clientes). |
| `worker2_redaccion_email.json` | Sub-workflow Worker 2 (redacción de emails). |
| `Reporte_Evidencias_Checkpoint4_SantiagoPacheco.pdf` | Reporte de evidencias: tests de ejecución con capturas, tabla de resultados y mapeo con la rúbrica. |

> El Manager llama a los Workers con *Execute Workflow* por ID. Al importar, hay que importar primero los Workers y reapuntar los dos nodos `Llamar Worker 1/2` a los workflows importados.

## Arquitectura

```
RAMA M4 (nueva)
Gmail Trigger ─► ¿Es auto-reply? ─┬─ sí ─► Ignorar auto-reply (corta el bucle)
                                   └─ no ─► Agente de soporte (LLM + Structured Output Parser)
                                            ─► Limpiar y validar payload (Set)
                                            ─► ¿Email válido? ─┬─ no ─► Descartar: sin email válido
                                                               └─ sí ─► Buscar contacto en HubSpot
                                                                        ─► ¿Contacto ya existe? ─┬─ sí ─► Actualizar contacto
                                                                                                  └─ no ─► Crear contacto
                                                                        ─► Crear borrador para aprobación (Gmail Draft)
                                                                        ─► Preparar aviso para Slack (Set)
                                                                        ─► Avisar al equipo en Slack

RAMA M1–M3 (sin cambios)
Chat Trigger ─► memoria Airtable ─► detección de nombre ─► Text Classifier ─► Worker 1 / Worker 2
             ─► trazabilidad por Gmail / escalamiento ─► historial + resumen automático
```

## Mapeo con la rúbrica

| # | Requisito | Nodo(s) | Cómo se cumple |
|---|---|---|---|
| ① | Filtro anti auto-reply justo después del Gmail Trigger | `¿Es auto-reply?` | IF con 8 condiciones en OR, *ignore case*: `auto-reply`, `automatic reply`, `out of office`, `undeliverable`, `respuesta automática`, `no-reply@`, `noreply@` y `Reporte del Manager` (evita que el sistema se lea a sí mismo). La rama true termina en un NoOp: se corta el bucle infinito. |
| ② | Look up before Create (evitar 409) | `Buscar contacto en HubSpot` → `¿Contacto ya existe?` → `Crear` / `Actualizar` | Se busca por email (`limit 1`, *Always Output Data*). Si hay `id` se actualiza, si no se crea. Probado: crear y luego actualizar sin duplicados. |
| ③ | Human-in-the-loop | `Crear borrador para aprobación` | Gmail **Create Draft**: nunca envía solo; una persona revisa y envía. |
| ④ | Set que limpia/valida el payload (evitar 400) | `Limpiar y validar payload` + `¿Email válido?` | Normaliza `email, nombre, asunto, respuesta, mensaje_cliente, categoria, threadId` con nombres limpios; si el email no es válido, se descarta antes de llamar al CRM. Un segundo Set (`Preparar aviso para Slack`) arma el texto mínimo para Slack. |

## Agente de soporte

- Modelo: `gpt-4o-mini` con *Structured Output Parser* (`categoria`, `respuesta`), `maxIterations = 5`.
- Taxonomía cerrada: `ESTADO_PEDIDO`, `DEVOLUCION_O_RECLAMO`, `CONSULTA_PRODUCTO`, `OTRO` (escape).
- El correo del cliente va entre `[INICIO DEL CORREO DEL CLIENTE]` / `[FIN DEL CORREO DEL CLIENTE]` como defensa contra *prompt injection*.

## Seguridad y mínimo privilegio

- **Credenciales por OAuth2** (HubSpot, Gmail, Slack) y API token para Airtable/OpenAI. **El JSON no contiene tokens ni secretos**, solo los IDs internos de credenciales de n8n.
- Permisos usados (mínimo necesario):
  - **HubSpot:** solo buscar y crear/actualizar contactos.
  - **Slack:** solo publicar mensajes (`chat:write`) en un canal.
  - **Gmail:** leer bandeja de entrada y crear borradores (no hay envío automático al cliente).
- En n8n Cloud, el botón *Connect my account* fija los scopes; por eso el mínimo privilegio se documenta aquí y se respeta en el diseño (el workflow solo usa esas operaciones).
- Aviso de privacidad: el JSON incluye mi email de pruebas, IDs de base/tabla de Airtable y el ID de un canal de Slack de práctica. No dan acceso por sí solos.

## Cómo importar y probar

1. En n8n: *Workflows → Import from file* y cargar primero los dos Workers, luego `checkpoint4_santiago_pacheco.json`.
2. Reasignar credenciales (HubSpot, Gmail, Slack, Airtable, OpenAI) y reapuntar `Llamar Worker 1/2`.
3. Mantener el workflow **inactivo** mientras se prueba (el Gmail Trigger hace polling cada minuto) y ejecutar con *Execute workflow*.

### Plan de pruebas (regresión manual)

Las capturas de cada ejecución están en `Reporte_Evidencias_Checkpoint4_SantiagoPacheco.pdf`.


| Prueba | Entrada | Resultado esperado |
|---|---|---|
| Auto-reply | Mail con asunto "Out of office" / remitente `no-reply@` | Sale por `Ignorar auto-reply`; no se llama al agente. |
| Mail real | Consulta de pedido de un cliente | Categoría `ESTADO_PEDIDO`, contacto creado en HubSpot, borrador en Gmail, aviso en Slack. |
| Mismo cliente otra vez (esperar ~30 s) | Segundo mail del mismo remitente | Rama `Actualizar contacto`; sin duplicado ni error 409. |
| Email inválido | Payload sin `@` | Sale por `Descartar: sin email válido`. |
| Regresión M3 | Mensaje por Chat Trigger | La rama de chat sigue respondiendo, guardando memoria y escalando como antes. |

## Limitaciones conocidas

- El nodo HubSpot ofrece una sola operación *Create/Update a contact*, por eso la lógica de lookup está explícita con IF.
- El índice de búsqueda de HubSpot tarda unos segundos en reflejar altas recientes: entre pruebas conviene esperar ~30 s.
- El borrador queda en Gmail y requiere envío manual (decisión de diseño de Human-in-the-loop).

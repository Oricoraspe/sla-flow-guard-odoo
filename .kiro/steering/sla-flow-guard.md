---
inclusion: always
---

# Steering del proyecto SLA Flow Guard

## Contexto del producto

SLA Flow Guard es un módulo nativo de Odoo que automatiza de forma controlada la clasificación de tickets autoasignados en la etapa configurada de **Falla Externa**. Su propósito es evitar consumo de SLA por un movimiento manual olvidado cuando existe una dependencia real de un tercero, sin pausar tickets que continúan bajo responsabilidad del equipo interno.

## Forma de comunicación

- Responde y redacta textos visibles en español claro.
- Usa identificadores de código en inglés y nombres que expresen intención.
- Explica primero el impacto funcional en tickets y SLA; después, el detalle técnico.
- Distingue entre comportamiento confirmado de la instancia, supuesto y propuesta.
- No inventes modelos, campos ni capacidades de Odoo. Confirma versión, edición y personalizaciones cuando sean relevantes.
- Si una solicitud pausaría SLA sin una dependencia externa válida, señala el riesgo y ofrece una alternativa auditable.

## Regla fundamental

**Nunca muevas automáticamente un ticket a Falla Externa solo porque fue asignado o porque el analista comenzó a atenderlo.** Debe coincidir con una regla activa que contenga al menos una condición funcional de dependencia externa, como categoría, servicio, etiqueta, origen o proveedor.

La única excepción es un equipo dedicado exclusivamente a fallas externas, documentado y aprobado por un coordinador. Incluso en ese caso se conservan confirmación, seguimiento y auditoría.

## Principios del producto

1. Odoo sigue siendo la autoridad sobre tickets, etapas y cálculo SLA.
2. El módulo cambia la etapa; no modifica fechas objetivo ni fórmulas SLA.
3. Toda transición automática debe poder explicarse mediante una regla y condiciones coincidentes.
4. Antes de cambiar la etapa se guarda el estado necesario para una restauración segura.
5. Los reintentos no pueden generar transiciones, actividades ni eventos duplicados.
6. Un cambio manual posterior tiene prioridad sobre la automatización.
7. Toda falla externa requiere confirmación, próxima revisión y trazabilidad.
8. Una regla nueva comienza en modo simulación antes de poder ejecutar movimientos.
9. No borres ni reescribas eventos históricos para mejorar indicadores.

## Stack y arquitectura obligatorios

- Implementa un add-on nativo de Odoo con Python, ORM y vistas XML.
- Extiende modelos mediante `_inherit`; no modifiques archivos del núcleo ni uses monkey patches.
- Usa PostgreSQL exclusivamente mediante el ORM de Odoo, salvo una necesidad excepcional documentada y revisada.
- Reutiliza `mail.activity`, `mail.thread`, grupos, ACL, reglas de registro e `ir.cron`.
- Añade JavaScript/OWL solo si las vistas estándar no resuelven el requisito.
- No introduzcas una aplicación web, base de datos, cola o servicio externo para esta funcionalidad sin una necesidad demostrable.
- Mantén separadas la detección del evento, evaluación de reglas, transición, interacción del usuario y auditoría.

## Organización del código

- `models/helpdesk_ticket.py`: campos y puntos de integración con el ticket.
- `models/sla_flow_rule.py`: configuración y validación de reglas.
- `models/sla_flow_event.py`: auditoría inmutable.
- `services/rule_evaluator.py`: evaluación pura sin escrituras.
- `services/transition_service.py`: único lugar para transiciones automáticas.
- `wizards/`: confirmación, rechazo y finalización de la dependencia.
- `data/`: tareas programadas, actividades y referencias instalables.
- `security/`: grupos, ACL y reglas multiempresa.
- `tests/`: reglas, transiciones, cron, seguridad e idempotencia.

No coloques reglas de negocio complejas en XML, métodos `onchange`, controladores o botones.

## Integración con tickets de Odoo

- Confirma el modelo y los campos reales antes de implementar; `helpdesk.ticket` es una referencia probable, no una suposición universal.
- Ejecuta detección en operaciones de servidor para cubrir interfaz, correo, importaciones y automatizaciones.
- Preserva el comportamiento de `super()` y soporta recordsets; no asumas que `create` o `write` recibe un único ticket.
- Usa `ensure_one()` solo en acciones que realmente operen sobre un único registro.
- Diferencia la asignación automática mediante un campo persistente o un contexto controlado por la automatización existente.
- Si no puede conocerse el origen, documenta la limitación y obliga a usar modo simulación antes de ampliar el disparador.
- Usa un contexto interno como `skip_sla_flow_guard=True` únicamente para evitar recursión de la transición propia.
- Ese contexto nunca debe omitir autorización, validaciones ni auditoría.

## Configuración de reglas

Cada regla debe incluir:

- nombre, compañía, equipo, secuencia, estado y versión;
- modo `simulation`, `confirmation` o `automatic`;
- al menos una condición funcional además de la asignación;
- etapa objetivo configurada como relación, no como ID numérico ni búsqueda por nombre;
- plazo de confirmación;
- intervalo de revisión y responsable de escalamiento;
- etapa de retorno opcional;
- creador, aprobador y fecha de activación.

Condiciones permitidas según disponibilidad real de la instancia:

- categoría o tipo;
- servicio afectado;
- etiquetas;
- canal u origen;
- proveedor;
- equipo;
- responsable o grupo;
- origen de asignación.

### Prioridad de reglas

- Filtra primero por compañía y equipo.
- Ordena por secuencia y luego por especificidad documentada.
- Una regla específica puede prevalecer sobre una general.
- Si dos reglas de igual prioridad producen acciones incompatibles, no muevas el ticket: crea una excepción de configuración.
- No ejecutes más de una transición para el mismo evento de asignación.

## Modos de actuación

### Simulación

- Evalúa y registra el resultado sin cambiar etapa ni crear una pausa.
- Úsalo obligatoriamente para validar una regla nueva con casos reales.

### Confirmación previa

- Crea una propuesta y actividad para el analista.
- No mueve a Falla Externa hasta recibir confirmación.
- Úsalo cuando una coincidencia requiera criterio humano.

### Automático

- Se reserva para reglas de alta certeza aprobadas.
- Guarda la etapa anterior, mueve una sola vez y crea confirmación posterior.
- Si falla una precondición, no efectúa una transición parcial.

## Estado interno

Estados válidos:

- `NOT_APPLICABLE`
- `PENDING_CONFIRMATION`
- `CONFIRMED`
- `REJECTED`
- `EXPIRED`
- `REVIEW_OVERDUE`
- `RESOLVED`
- `CONFLICT`

Reglas de transición:

- `PENDING_CONFIRMATION → CONFIRMED` exige motivo, proveedor y próxima revisión.
- `PENDING_CONFIRMATION → REJECTED` intenta restaurar solo si la etapa continúa siendo la aplicada por Flow Guard.
- Una confirmación vencida puede restaurarse conforme a la regla.
- Una revisión vencida de una falla confirmada crea actividad y escalamiento; no elimina silenciosamente la clasificación.
- `CONFIRMED → RESOLVED` registra el final y retorna a la etapa guardada o configurada.
- Cualquier diferencia entre el estado esperado y el actual produce `CONFLICT` sin sobrescribir el ticket.
- Centraliza estas transiciones; no las dupliques entre modelo, asistentes y cron.

## Confirmación y datos requeridos

Para confirmar exige:

- motivo de dependencia externa;
- proveedor o tercero;
- próxima fecha de revisión;
- usuario y fecha de confirmación;
- referencia externa cuando esté disponible.

No exijas una referencia que el proveedor todavía no haya entregado; permite agregarla posteriormente y registra el cambio.

Antes de restaurar o finalizar, compara etapa, responsable, estado interno y marca de actualización esperados. Si hubo una acción humana posterior, conserva esa acción y registra conflicto.

## Idempotencia y auditoría

- Genera una identidad estable por ticket, evento de asignación, regla y versión.
- Reutiliza el resultado si la misma identidad vuelve a procesarse.
- Una actividad de confirmación o revisión debe tener una clave que impida duplicados.
- Registra ticket, compañía, regla, condiciones coincidentes, etapas, estados, actor, fechas y resultado.
- El historial es inmutable para usuarios normales: bloquea edición y eliminación desde el flujo habitual.
- Una corrección crea un evento compensatorio, no modifica el evento original.
- No publiques datos sensibles en el chatter si solo corresponden al registro técnico de auditoría.

## Procesos programados

- Procesa confirmaciones y revisiones en lotes limitados.
- Usa dominios indexables y evita recorrer todos los tickets.
- Continúa con otros registros cuando uno produce un error controlado.
- Ejecutar el cron varias veces debe producir el mismo estado final y no duplicar actividades.
- Prioriza confirmaciones vencidas antes de revisiones ordinarias.
- Registra cantidades procesadas, restauradas, escaladas, omitidas y en conflicto.
- No restaure un ticket confirmado solo por una falla temporal del cron.

## Seguridad y multiempresa

- Define grupos separados para usuario, coordinador, administrador y auditor.
- Respeta los permisos existentes sobre el ticket antes de permitir acciones de Flow Guard.
- Filtra reglas, eventos y reportes por compañías autorizadas.
- Verifica que las etapas configuradas pertenezcan a la compañía o equipo compatible.
- No uses `sudo()` como solución general; si una tarea técnica lo requiere, reduce su alcance y conserva actor funcional y auditoría.
- Valida permisos en métodos Python aunque el botón esté oculto en XML.
- El auditor tiene acceso de lectura y no puede provocar transiciones.

## Interfaz y accesibilidad

- Todo texto visible debe estar en español.
- Muestra claramente si el resultado es simulación, propuesta, movimiento automático o conflicto.
- En el ticket presenta estado de Flow Guard, regla, etapa anterior, proveedor y próxima revisión.
- Los asistentes deben explicar el efecto sobre la etapa antes de confirmar.
- No dependas únicamente de color para distinguir estados.
- Oculta campos técnicos cuando no aporten valor, pero mantén acceso al historial para roles autorizados.
- Confirma acciones que finalizan o revierten una dependencia.

## Calidad del código

- Usa métodos pequeños, retornos tempranos y nombres explícitos.
- Prefiere modelos y servicios del framework sobre SQL o automatizaciones duplicadas.
- Usa campos computados almacenados solo cuando exista una necesidad de búsqueda o rendimiento.
- Declara dependencias, datos y seguridad en `__manifest__.py` en orden consistente.
- Usa referencias XML estables para datos creados por el módulo.
- No fijes IDs de base de datos, nombres traducibles de etapas, usuarios ni compañías en código.
- Evita efectos secundarios dentro del evaluador de reglas.
- Agrega logs técnicos con ticket, regla y correlación, sin contenido sensible innecesario.

## Validación de cambios futuros

Para cambios de comportamiento, valida como mínimo:

1. ticket coincidente en modo automático;
2. ticket sin coincidencia que conserva etapa y SLA normal;
3. regla en simulación que no escribe;
4. confirmación y rechazo;
5. vencimiento procesado dos veces sin duplicados;
6. cambio manual posterior que produce conflicto;
7. bucle de recursión evitado;
8. usuario sin permiso;
9. acceso desde otra compañía;
10. evento de auditoría no editable.

Antes de afirmar que un cambio está terminado, ejecuta las comprobaciones disponibles para la versión de Odoo, documenta supuestos no verificados y resume archivos, validaciones y riesgos pendientes.

# Actividad: definición de un producto tecnológico

## 1. Caso de uso elegido

**Automatización controlada del paso a “Falla Externa” para tickets autoasignados en Odoo.**

### Situación actual

La empresa utiliza Odoo Helpdesk y ya tiene configurados los SLA. Cuando un ticket entra y se asigna automáticamente a un analista, puede corresponder a una falla o dependencia de un proveedor externo. En esos casos debe pasar a la etapa **Falla Externa**, donde Odoo pausa el consumo del SLA de acuerdo con la configuración existente.

El movimiento depende actualmente de una acción manual. Si el analista empieza a gestionar el caso y olvida cambiar la etapa, el SLA continúa consumiéndose aunque la resolución dependa legítimamente de un tercero. El resultado puede ser un incumplimiento que no representa el desempeño real del equipo.

### Precisión importante

La etapa **Falla Externa** solo debe utilizarse cuando la atención está bloqueada por un tercero. No debe aplicarse automáticamente a cualquier ticket mientras el analista trabaja, porque eso pausaría SLA válidos y distorsionaría los indicadores. La solución automatiza únicamente casos que cumplen reglas explícitas y conserva evidencia de cada decisión.

## 2. Definición del producto

### Nombre

**SLA Flow Guard**

### Descripción

Módulo nativo de Odoo que detecta tickets autoasignados con dependencia externa, evalúa reglas configurables, los mueve de forma segura a la etapa **Falla Externa** y administra confirmación, seguimiento, restauración y auditoría.

### Objetivo general

Evitar el consumo incorrecto de SLA causado por movimientos manuales olvidados, sin pausar tickets que realmente deben permanecer en atención activa.

### Propuesta de valor

- Reduce tareas manuales y errores de clasificación.
- Protege los indicadores SLA sin alterar sus reglas ni fechas.
- Conserva la etapa anterior para retornar el ticket correctamente.
- Exige motivo y trazabilidad para cada pausa.
- Recuerda al analista revisar periódicamente la dependencia externa.
- Evita que un ticket permanezca indefinidamente pausado.

### Lo que hace y lo que no hace

| SLA Flow Guard hace | SLA Flow Guard no hace |
|---|---|
| Cambia de etapa según reglas verificables. | Pausa todos los tickets recién asignados. |
| Utiliza la etapa y el cálculo SLA ya configurados en Odoo. | Recalcula o modifica las fechas objetivo del SLA. |
| Registra regla, motivo, usuario y duración. | Oculta pausas o elimina historial. |
| Devuelve el ticket a su etapa operativa. | Cierra o resuelve automáticamente el problema técnico. |
| Solicita confirmación en casos configurados. | Sustituye el criterio profesional del analista. |

## 3. Usuarios y roles

| Rol | Responsabilidades |
|---|---|
| Analista TI | Confirmar o rechazar la falla externa, indicar proveedor, registrar referencia y reanudar la atención. |
| Coordinador de soporte | Definir reglas, revisar excepciones y controlar tickets pausados durante demasiado tiempo. |
| Administrador de Odoo | Configurar etapas, permisos, compañías, actividades programadas e instalación del módulo. |
| Auditor o responsable de servicio | Consultar el historial y verificar que las pausas tengan una causa válida. |

## 4. Escenario de ejemplo

1. Odoo recibe el ticket `TK-1842` desde una alerta de conectividad.
2. La automatización existente lo asigna a una analista del equipo de Infraestructura.
3. SLA Flow Guard detecta el cambio de “sin responsable” a “responsable asignado”.
4. Evalúa las reglas activas y encuentra esta coincidencia:
   - equipo: Infraestructura;
   - categoría: Conectividad;
   - etiqueta: Proveedor ISP;
   - origen: Monitor de enlaces;
   - modo: movimiento automático.
5. Guarda la etapa original **Nuevo**, mueve el ticket a **Falla Externa** y registra la regla aplicada.
6. Crea una actividad con vencimiento en 15 minutos para confirmar proveedor y motivo.
7. La analista confirma que el enlace del proveedor está caído e ingresa el número de caso externo.
8. El módulo mantiene la etapa y programa una revisión cada dos horas.
9. Cuando el proveedor restablece el servicio, la analista pulsa **Finalizar dependencia externa**.
10. El ticket vuelve a **En progreso**, Odoo reanuda su flujo SLA normal y el evento queda auditado.

### Escenario alternativo

Si la analista rechaza la clasificación o vence el tiempo de confirmación, el módulo devuelve el ticket a la etapa guardada. Si el ticket fue modificado manualmente entretanto, no sobrescribe el cambio: genera una excepción para revisión.

## 5. Funcionamiento del producto

### 5.1 Detección de asignación

El módulo observa la creación del ticket y los cambios del responsable. Para distinguir una asignación automática de una manual se utilizará una marca de origen o contexto proporcionada por la automatización existente. Si la instalación actual no expone esa información, se deberá añadir el campo `assignment_source` o configurar la regla para actuar ante cualquier primera asignación del equipo seleccionado.

No se asumirá que Odoo identifica de forma estándar el origen de toda asignación; este punto debe confirmarse en la versión y personalizaciones de la empresa.

### 5.2 Evaluación de reglas

Cada regla puede utilizar:

- compañía;
- equipo de soporte;
- categoría o tipo de ticket;
- servicio afectado;
- una o varias etiquetas;
- canal u origen;
- proveedor externo;
- responsable o grupo de responsables;
- tipo de asignación;
- prioridad y secuencia de evaluación.

Las reglas se evalúan por prioridad. Una regla más específica prevalece sobre una general. Si dos reglas con la misma prioridad producen resultados incompatibles, el módulo no mueve el ticket y crea una alerta de configuración.

### 5.3 Modos de actuación

- **Modo automático:** reservado para reglas de alta certeza. Mueve inmediatamente el ticket y solicita confirmación posterior.
- **Modo con confirmación previa:** propone el movimiento al analista, pero no pausa el SLA hasta que lo confirme.
- **Modo simulación:** registra qué habría hecho sin cambiar la etapa. Se utiliza antes de activar una regla nueva.

### 5.4 Confirmación de falla externa

La confirmación registra como mínimo:

- motivo de la dependencia;
- proveedor o tercero responsable;
- fecha y usuario que confirma;
- próxima fecha de revisión;
- referencia externa, cuando esté disponible.

La confirmación no puede alterar retroactivamente el historial.

### 5.5 Seguimiento y restauración

- Crea actividades de revisión mientras la dependencia continúa.
- Escala al coordinador cuando vence la revisión o se supera un límite configurable.
- Permite finalizar la dependencia y retornar a la etapa anterior o a una etapa configurada.
- No restaura automáticamente si una persona ya cambió el ticket a otra etapa.
- Evita ciclos de automatización mediante una marca interna de contexto e idempotencia.

## 6. Estados internos de control

| Estado | Significado |
|---|---|
| `NOT_APPLICABLE` | El ticket no cumple una regla de falla externa. |
| `PENDING_CONFIRMATION` | Se aplicó o propuso una regla y espera validación del analista. |
| `CONFIRMED` | La dependencia externa fue confirmada y está en seguimiento. |
| `REJECTED` | El analista indicó que no corresponde a una falla externa. |
| `EXPIRED` | Venció la confirmación o revisión y requiere restauración o escalamiento. |
| `RESOLVED` | Terminó la dependencia y el ticket regresó al flujo operativo. |
| `CONFLICT` | El ticket cambió manualmente y la automatización no debe sobrescribirlo. |

Estos estados complementan la etapa de Odoo; no sustituyen las etapas del equipo de soporte.

## 7. Requisitos funcionales

| ID | Requisito |
|---|---|
| RF-01 | Detectar la primera asignación o una asignación marcada como automática. |
| RF-02 | Evaluar reglas activas según compañía, equipo y condiciones configuradas. |
| RF-03 | Permitir modos automático, confirmación previa y simulación. |
| RF-04 | Guardar la etapa anterior antes de cualquier transición automática. |
| RF-05 | Mover a la etapa configurada de Falla Externa sin usar identificadores de base de datos fijos. |
| RF-06 | Crear una actividad de confirmación con plazo configurable. |
| RF-07 | Exigir motivo y proveedor para confirmar la dependencia. |
| RF-08 | Restaurar la etapa al rechazar, vencer o finalizar la dependencia. |
| RF-09 | Respetar cambios manuales posteriores y registrar conflictos. |
| RF-10 | Crear recordatorios y escalamiento para pausas prolongadas. |
| RF-11 | Mantener un historial inmutable de evaluación, transición y confirmación. |
| RF-12 | Mostrar reportes de pausas por regla, equipo, proveedor, duración y resultado. |
| RF-13 | Aplicar permisos por rol, compañía y equipo. |
| RF-14 | Evitar que reintentos o eventos repetidos generen movimientos duplicados. |

## 8. Reglas de negocio

1. Un ticket solo puede entrar automáticamente a **Falla Externa** si coincide con una regla activa y autorizada.
2. Estar asignado o en atención no constituye por sí solo una falla externa.
3. El módulo cambia la etapa; Odoo continúa siendo la autoridad sobre el cálculo SLA.
4. La etapa objetivo se configura por referencia; nunca se busca únicamente por el texto “Falla Externa”.
5. Antes de mover se conserva etapa, fecha y versión del ticket.
6. Todo movimiento automático registra regla, condiciones coincidentes y usuario técnico.
7. La confirmación requiere motivo, proveedor y próxima revisión.
8. Si vence una confirmación, se restaura la etapa salvo que exista un cambio manual posterior.
9. Si vence una revisión confirmada, se notifica y escala; no se altera silenciosamente el ticket.
10. Desactivar o eliminar una regla no modifica retroactivamente los tickets procesados.
11. Una regla nueva debe pasar primero por modo simulación antes de habilitar movimientos automáticos.
12. Los usuarios sin permiso no pueden confirmar, restaurar ni modificar reglas.

## 9. Requisitos no funcionales

- **Seguridad:** controles de acceso y reglas de registro por compañía y equipo.
- **Trazabilidad:** todos los movimientos automáticos y decisiones quedan registrados.
- **Rendimiento:** la evaluación normal no debe retrasar perceptiblemente la creación o asignación del ticket.
- **Resiliencia:** los recordatorios se ejecutan por lotes y pueden reintentarse sin duplicar actividades.
- **Compatibilidad:** no se modifica el código base de Odoo; se utiliza herencia del modelo y referencias configurables.
- **Usabilidad:** el analista puede confirmar o rechazar desde el formulario del ticket con pocos campos obligatorios.
- **Mantenibilidad:** la evaluación de reglas está separada de la transición y de las vistas.
- **Multiempresa:** reglas, etapas, permisos y reportes respetan la compañía activa.
- **Auditabilidad:** no se permite editar ni eliminar eventos desde el flujo habitual.

## 10. Alcance del MVP

- Un módulo instalable en la versión de Odoo utilizada por la empresa.
- Reglas por compañía, equipo, categoría, etiqueta y tipo de asignación.
- Modos simulación, automático y confirmación previa.
- Movimiento a Falla Externa y conservación de etapa anterior.
- Asistente de confirmación y finalización.
- Actividades de recordatorio y proceso programado de vencimientos.
- Registro de auditoría y vista básica de excepciones.

### Evolución posterior

- Reglas por proveedor, servicio y horario.
- Confirmación desde notificaciones interactivas.
- Métricas de duración externa y recurrencia por proveedor.
- Sugerencias basadas en patrones históricos, siempre sujetas a validación.
- Doble aprobación para pausas superiores a un límite.

### Fuera de alcance

- Cambiar fechas objetivo o políticas SLA.
- Pausar todos los tickets asignados.
- Resolver incidentes o contactar automáticamente al proveedor.
- Modificar el núcleo de Odoo.
- Borrar o reescribir el historial de pausas.

## 11. Historia de usuario principal

> Como analista TI, quiero que los tickets autoasignados que cumplen criterios de falla externa pasen automáticamente a la etapa correcta y me soliciten confirmación, para no consumir SLA por un olvido y mantener evidencia de la causa.

### Criterios de aceptación

- Dado un ticket recién autoasignado que coincide con una regla automática activa, el sistema guarda su etapa anterior y lo mueve una sola vez a Falla Externa.
- El ticket muestra la regla aplicada, el estado `PENDING_CONFIRMATION` y la fecha límite de confirmación.
- Se crea una única actividad para la analista responsable.
- Al confirmar, se exigen motivo, proveedor y próxima revisión.
- Al rechazar dentro del plazo, el ticket vuelve a la etapa anterior y queda un evento de rechazo.
- Al vencer sin confirmación, el proceso programado restaura el ticket si no hubo cambios manuales.
- Si otra persona cambió la etapa, el módulo no la sobrescribe y registra `CONFLICT`.
- Un ticket sin regla coincidente permanece en su etapa y SLA normal.
- Cada resultado puede rastrearse hasta la regla y condiciones evaluadas.

## 12. Indicadores de éxito

- Reducción de tickets con dependencia externa que consumen SLA por clasificación tardía.
- Porcentaje de movimientos automáticos confirmados frente a rechazados.
- Cantidad de falsos positivos por regla.
- Tiempo promedio entre asignación y clasificación correcta.
- Cantidad de pausas vencidas o sin revisión.
- Duración de dependencias por proveedor y categoría.
- Número de movimientos duplicados: objetivo igual a cero.
- Porcentaje de eventos con trazabilidad completa: objetivo 100 %.

## 13. Riesgos y mitigaciones

| Riesgo | Mitigación |
|---|---|
| Pausar tickets que no son fallas externas | Reglas específicas, modo simulación, confirmación y seguimiento de falsos positivos. |
| Reglas contradictorias | Prioridad, validación de solapamiento y bloqueo ante empate incompatible. |
| Bucle al cambiar la etapa | Contexto técnico de exclusión e idempotencia por ticket y evento. |
| Restaurar sobre un cambio manual | Comparar etapa y versión antes de escribir; registrar conflicto. |
| Ticket pausado indefinidamente | Fecha de revisión, actividades y escalamiento. |
| Nombre o ID de etapa distinto por compañía | Configuración por compañía y referencias, sin IDs fijos. |
| Diferencias entre versiones de Odoo | Confirmar modelos y capacidades antes de implementar el módulo. |
| Uso para ocultar incumplimientos | Auditoría inmutable, permisos y regla expresa de dependencia externa real. |

## 14. Solución técnica resumida

SLA Flow Guard se implementará como un módulo nativo de Odoo con Python, ORM de Odoo, vistas XML, actividades y tareas programadas. Extenderá el ticket de Helpdesk sin modificar el núcleo. La evaluación, la transición y la auditoría estarán separadas para facilitar pruebas y mantenimiento. El stack y la estructura detallados se encuentran en [arquitectura.md](arquitectura.md).

El steering permanente [`.kiro/steering/sla-flow-guard.md`](../.kiro/steering/sla-flow-guard.md) definirá reglas específicas para evitar pausas injustificadas, IDs fijos, bucles de automatización y pérdida de trazabilidad.

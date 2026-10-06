# Stack tecnológico y arquitectura de SLA Flow Guard

## 1. Enfoque de solución

SLA Flow Guard se implementará como un **módulo nativo de Odoo** instalado junto a Helpdesk. Extenderá el modelo de tickets mediante herencia, sin modificar el núcleo de Odoo y sin crear una aplicación paralela.

Este enfoque es apropiado porque:

- el disparador es un cambio realizado dentro de Odoo;
- las etapas, usuarios, equipos y SLA ya existen en Odoo;
- las actividades y permisos pueden reutilizar la plataforma;
- la transición debe participar en la misma operación del ticket;
- se evita mantener una API, base de datos y sistema de autenticación adicionales.

El módulo **no calcula ni pausa directamente el SLA**. Cambia el ticket a la etapa configurada y Odoo aplica las reglas SLA que la organización ya tiene asociadas con esa etapa.

## 2. Supuestos por confirmar antes de implementar

- Versión y edición exactas de Odoo.
- Nombre técnico del módulo Helpdesk instalado y personalizaciones existentes.
- Modelo real del ticket, normalmente `helpdesk.ticket`.
- Campo utilizado para responsable, equipo, categoría, etiquetas y etapa.
- Forma en que la automatización actual asigna el ticket.
- Cómo identifica Odoo que la etapa **Falla Externa** pausa el SLA.
- Si existe una etapa diferente por compañía o equipo.
- Calendarios, permisos y reglas multiempresa.

No se desarrollará una integración basándose únicamente en nombres visibles, porque estos pueden cambiar por idioma o configuración.

## 3. Stack tecnológico

| Capa | Tecnología | Uso |
|---|---|---|
| Plataforma | Odoo | Ejecución, modelos, vistas, seguridad, actividades y tareas programadas. |
| Lenguaje | Python | Reglas de evaluación, transiciones, validaciones y auditoría. |
| Persistencia | PostgreSQL mediante ORM de Odoo | Configuración, estado de control y eventos, sin SQL directo. |
| Interfaz | Vistas XML de Odoo | Formularios, listas, menús, búsquedas y asistentes. |
| Interacciones opcionales | OWL/JavaScript de Odoo | Avisos dinámicos, solo si las vistas estándar no son suficientes. |
| Mensajería interna | `mail.thread` y `mail.activity` | Registro visible, confirmaciones, recordatorios y escalamiento. |
| Procesos programados | `ir.cron` | Confirmaciones vencidas y revisiones pendientes por lotes. |
| Seguridad | Grupos, ACL y reglas de registro de Odoo | Acceso por rol, equipo y compañía. |
| Pruebas | Framework de pruebas de Odoo | Reglas, transiciones, permisos, cron e idempotencia. |
| Calidad Python | Ruff y Pylint con reglas para Odoo | Formato, errores frecuentes y convenciones del framework. |
| Entorno | Docker, si el proyecto ya lo utiliza | Instancia reproducible de desarrollo con Odoo y PostgreSQL. |

Las versiones de Python y herramientas deben ser compatibles con la versión instalada de Odoo. No se añadirán dependencias externas si el framework ya ofrece la capacidad necesaria.

## 4. Vista de componentes

```text
┌──────────────────────────────── Odoo ────────────────────────────────┐
│                                                                      │
│  Automatización de asignación                                        │
│              │                                                       │
│              ▼                                                       │
│  Extensión de helpdesk.ticket                                        │
│              │ evento de primera asignación                          │
│              ▼                                                       │
│  Evaluador de reglas ───────► Modo simulación / conflicto            │
│              │ coincidencia válida                                   │
│              ▼                                                       │
│  Servicio de transición                                               │
│      ├── guarda etapa anterior                                       │
│      ├── mueve a Falla Externa                                       │
│      ├── crea evento de auditoría                                    │
│      └── crea actividad de confirmación                              │
│              │                                                       │
│              ▼                                                       │
│  Asistente del analista ─────► Confirmar / rechazar / finalizar       │
│              │                                                       │
│              ▼                                                       │
│  ir.cron ────────────────────► Vencimientos, recordatorios, escalado  │
│                                                                      │
└──────────────────────── PostgreSQL vía ORM ──────────────────────────┘
```

## 5. Componentes del módulo

### 5.1 Extensión del ticket

Extiende el ticket de Helpdesk para:

- detectar creación o cambio de responsable;
- almacenar el origen de asignación cuando sea necesario;
- mostrar el estado interno de Flow Guard;
- relacionar la regla aplicada;
- conservar etapa y versión previas;
- mostrar proveedor, referencia, fechas y próxima revisión;
- ofrecer acciones de confirmar, rechazar y finalizar dependencia.

La lógica no dependerá de `onchange`, porque este solo se ejecuta en determinados flujos de interfaz. Las reglas deben aplicarse también a importaciones, acciones programadas y operaciones del servidor.

### 5.2 Evaluador de reglas

Recibe una instantánea del ticket y devuelve una decisión sin modificar registros:

```text
NO_MATCH
PROPOSE_TRANSITION
APPLY_TRANSITION
CONFIGURATION_CONFLICT
```

Sus responsabilidades son:

- filtrar reglas por compañía y equipo;
- comprobar condiciones configuradas;
- ordenar por secuencia y especificidad;
- detectar resultados incompatibles;
- explicar qué condiciones coincidieron;
- respetar los modos simulación, confirmación y automático.

Separar la evaluación permite probar las reglas sin cambiar tickets.

### 5.3 Servicio de transición

Es el único componente autorizado para cambiar automáticamente la etapa. Debe:

1. volver a leer el estado vigente;
2. comprobar que no hubo un cambio manual posterior;
3. guardar etapa anterior y marca de actualización;
4. aplicar la transición con un contexto que evite recursión;
5. crear el evento de auditoría;
6. crear una única actividad de confirmación;
7. devolver un resultado explícito.

Si cualquiera de las verificaciones falla, no debe sobrescribir el ticket.

### 5.4 Asistentes

Se proponen tres asistentes de Odoo:

- **Confirmar falla externa:** solicita motivo, proveedor, referencia y próxima revisión.
- **Rechazar clasificación:** solicita motivo y propone restaurar la etapa anterior.
- **Finalizar dependencia externa:** registra resultado y devuelve el ticket a la etapa anterior o a una etapa configurada.

Los asistentes aplican permisos y validaciones en el servidor, no solo en la vista.

### 5.5 Proceso programado

Un `ir.cron` procesa por lotes:

- confirmaciones vencidas;
- revisiones próximas o atrasadas;
- actividades faltantes;
- excepciones pendientes de escalamiento.

El proceso debe ser idempotente: ejecutarlo dos veces no crea dos actividades ni restaura dos veces un ticket.

### 5.6 Auditoría y reportes

Un modelo inmutable registra:

- ticket y compañía;
- fecha y tipo de evento;
- regla y condiciones coincidentes;
- etapa anterior y nueva;
- estado antes y después;
- usuario o proceso responsable;
- motivo, proveedor y referencia externa;
- identificador de correlación;
- resultado o conflicto.

Los reportes se enfocan en calidad del proceso: confirmaciones, rechazos, falsos positivos, pausas vencidas y duración por proveedor.

## 6. Detección del origen de asignación

Odoo puede recibir asignaciones desde distintos mecanismos. No se debe asumir que el método de asignación está disponible automáticamente en todos los casos.

### Opción recomendada

La automatización existente escribe un campo de selección como:

```text
assignment_source = manual | automatic_rule | email_alias | monitoring | import
```

Esto permite auditar el origen incluso después de terminada la transacción.

### Opción alternativa

Si la automatización se ejecuta en la misma operación y puede controlarse, utiliza un contexto explícito como `assignment_source=automatic_rule`. Esta opción no conserva evidencia por sí sola, por lo que el evento de Flow Guard debe copiar el valor.

### Compatibilidad

Si no es posible modificar la automatización existente, una regla puede activarse ante la primera asignación de equipos específicos. Este modo es menos preciso y debe probarse primero en simulación.

## 7. Algoritmo de evaluación

```text
1. Ignorar si el contexto indica una transición interna de Flow Guard.
2. Ignorar si no ocurrió una asignación relevante.
3. Cargar reglas activas de la compañía y el equipo.
4. Evaluar condiciones sin modificar el ticket.
5. Ordenar coincidencias por secuencia y especificidad.
6. Si no hay coincidencia: registrar solo si está habilitado el diagnóstico.
7. Si existe empate incompatible: crear excepción y no mover.
8. Si la regla está en simulación: registrar resultado y no mover.
9. Si requiere confirmación previa: crear propuesta y actividad.
10. Si es automática: validar versión y ejecutar una transición idempotente.
```

Una regla nunca debe activarse solo porque existe un responsable. Debe existir al menos una condición funcional relacionada con una dependencia externa, salvo que el equipo completo gestione exclusivamente ese tipo de casos y dicha excepción esté documentada.

## 8. Modelo de datos propuesto

### Extensión de `helpdesk.ticket`

| Campo conceptual | Propósito |
|---|---|
| `flow_guard_state` | Estado interno de la automatización. |
| `flow_guard_rule_id` | Regla que originó la última decisión. |
| `flow_guard_previous_stage_id` | Etapa guardada para una restauración segura. |
| `flow_guard_assignment_source` | Origen de la asignación. |
| `flow_guard_started_at` | Inicio de la clasificación externa. |
| `flow_guard_confirmation_deadline` | Límite para confirmar. |
| `flow_guard_confirmed_at` | Fecha de confirmación. |
| `flow_guard_review_at` | Próxima revisión de la dependencia. |
| `flow_guard_provider_id` | Proveedor o tercero responsable. |
| `flow_guard_external_reference` | Número de caso o referencia externa. |
| `flow_guard_expected_write_date` | Versión esperada para detectar cambios posteriores. |

Los nombres definitivos deben ajustarse a las convenciones del módulo y evitar conflictos con campos existentes.

### Modelo `helpdesk.sla.flow.rule`

| Grupo de campos | Contenido |
|---|---|
| Identidad | nombre, activa, compañía, secuencia y descripción. |
| Alcance | equipo, responsables o grupos permitidos. |
| Condiciones | categorías, servicios, etiquetas, canal, proveedor y origen de asignación. |
| Acción | modo, etapa objetivo, plazo de confirmación y etapa de retorno opcional. |
| Seguimiento | intervalo de revisión, límite máximo y responsable de escalamiento. |
| Gobierno | creador, aprobador, fecha de activación y versión. |

La etapa objetivo se guarda como relación a una etapa permitida, nunca como un ID numérico fijo ni como una búsqueda por texto.

### Modelo `helpdesk.sla.flow.event`

Registro inmutable de evaluaciones y transiciones. Su eliminación y edición se bloquean para usuarios normales. Una corrección se representa mediante un evento adicional, no sobrescribiendo el anterior.

## 9. Estados y transiciones

```text
NOT_APPLICABLE

PENDING_CONFIRMATION ──► CONFIRMED ──► RESOLVED
          │                    │
          ├──► REJECTED        ├──► REVIEW_OVERDUE
          ├──► EXPIRED         └──► CONFLICT
          └──► CONFLICT
```

Reglas principales:

- `PENDING_CONFIRMATION → CONFIRMED` exige motivo, proveedor y próxima revisión.
- `PENDING_CONFIRMATION → REJECTED` intenta restaurar solo si la etapa no cambió.
- Una confirmación vencida puede restaurarse según configuración.
- Una revisión vencida no elimina una falla ya confirmada; crea recordatorio y escalamiento.
- `CONFIRMED → RESOLVED` registra fin de dependencia y retorna al flujo operativo.
- `CONFLICT` siempre requiere revisión humana.

## 10. Estructura propuesta

```text
proyecto-odoo/
├── .kiro/
│   └── steering/
│       └── sla-flow-guard.md
├── docs/
│   ├── actividad.md
│   └── arquitectura.md
├── addons/
│   └── sla_flow_guard/
│       ├── __init__.py
│       ├── __manifest__.py
│       ├── models/
│       │   ├── __init__.py
│       │   ├── helpdesk_ticket.py
│       │   ├── sla_flow_rule.py
│       │   └── sla_flow_event.py
│       ├── services/
│       │   ├── __init__.py
│       │   ├── rule_evaluator.py
│       │   └── transition_service.py
│       ├── wizards/
│       │   ├── __init__.py
│       │   ├── confirm_external_failure.py
│       │   ├── reject_external_failure.py
│       │   └── finish_external_failure.py
│       ├── views/
│       │   ├── helpdesk_ticket_views.xml
│       │   ├── sla_flow_rule_views.xml
│       │   ├── sla_flow_event_views.xml
│       │   └── sla_flow_wizard_views.xml
│       ├── security/
│       │   ├── ir.model.access.csv
│       │   ├── sla_flow_groups.xml
│       │   └── sla_flow_security.xml
│       ├── data/
│       │   ├── ir_cron.xml
│       │   ├── mail_activity_type.xml
│       │   └── sequence.xml
│       ├── reports/
│       │   └── sla_flow_report_views.xml
│       ├── tests/
│       │   ├── __init__.py
│       │   ├── test_rule_evaluator.py
│       │   ├── test_transitions.py
│       │   ├── test_scheduled_actions.py
│       │   └── test_security.py
│       └── static/
│           └── description/
│               └── index.html
└── README.md
```

## 11. Responsabilidad de cada directorio

- **`models/`** define campos, restricciones y puntos de integración con el ORM.
- **`services/`** contiene evaluación y transición sin depender de la interfaz.
- **`wizards/`** captura decisiones humanas y llama a servicios del dominio.
- **`views/`** presenta configuración, ticket, auditoría y excepciones.
- **`security/`** define grupos, accesos por modelo y reglas multiempresa.
- **`data/`** instala cron, tipos de actividad y referencias estables.
- **`reports/`** agrega vistas analíticas del proceso.
- **`tests/`** valida reglas, concurrencia, idempotencia, cron y permisos.

## 12. Seguridad y permisos

Se proponen estos grupos:

- **Usuario de Flow Guard:** ve el estado y confirma o rechaza en tickets permitidos.
- **Coordinador de Flow Guard:** gestiona reglas, revisiones y excepciones de sus equipos.
- **Administrador de Flow Guard:** configura etapas, compañías y parámetros técnicos.
- **Auditor de Flow Guard:** consulta eventos sin modificar tickets ni reglas.

Controles obligatorios:

- separar lectura, confirmación, gestión de reglas y auditoría;
- filtrar reglas y eventos por compañías autorizadas;
- respetar el acceso existente al ticket;
- no utilizar `sudo()` como solución general a errores de permisos;
- bloquear `write` y `unlink` normales sobre eventos de auditoría;
- validar permisos nuevamente en métodos de asistentes y servicios.

## 13. Idempotencia, recursión y concurrencia

### Idempotencia

Cada decisión automática utiliza una clave conceptual formada por ticket, evento de asignación, regla y versión. Si la misma operación vuelve a ejecutarse, devuelve el resultado existente.

### Prevención de recursión

El cambio interno de etapa utiliza un contexto como `skip_sla_flow_guard=True`. Este contexto solo evita reevaluar la misma transición; no omite permisos ni auditoría.

### Concurrencia

Antes de restaurar o cambiar etapa, el servicio compara:

- etapa actual;
- responsable actual;
- regla vigente;
- marca de actualización guardada;
- estado interno esperado.

Si alguno cambió, registra `CONFLICT` y no sobrescribe la acción humana.

## 14. Tareas programadas

El cron debe:

- seleccionar lotes limitados para no bloquear la instancia;
- procesar primero confirmaciones vencidas y luego revisiones;
- reservar o marcar cada elemento antes de actuar;
- usar una clave estable para actividades y evitar duplicados;
- registrar cantidad procesada, omitida, restaurada y en conflicto;
- continuar con otros tickets si uno falla;
- escalar errores repetidos al administrador.

## 15. Estrategia de implementación

1. **Descubrimiento:** confirmar versión, modelos, etapas, asignación y comportamiento real del SLA.
2. **Modelo y seguridad:** crear reglas, eventos, grupos y campos del ticket.
3. **Modo simulación:** ejecutar reglas sin modificar etapas y medir coincidencias/falsos positivos.
4. **Confirmación previa:** habilitar propuestas y asistentes antes del movimiento.
5. **Automatización controlada:** activar solo reglas con resultados confiables.
6. **Seguimiento:** añadir cron, recordatorios, restauración y conflictos.
7. **Reportes:** analizar precisión de reglas, duración externa y revisiones vencidas.

## 16. Validaciones técnicas previstas

- Una regla específica prevalece sobre una general.
- Reglas incompatibles no mueven el ticket.
- Un evento repetido no genera otra transición ni actividad.
- El contexto interno no produce un bucle de `write`.
- Una confirmación exige los datos definidos.
- Un rechazo restaura la etapa correcta.
- Un cambio manual posterior genera conflicto.
- El cron puede ejecutarse dos veces con el mismo resultado.
- Un usuario de otra compañía no ve ni modifica datos no autorizados.
- El evento de auditoría no puede editarse o eliminarse mediante el flujo normal.

## 17. Despliegue

- Instalar primero en una base de desarrollo con copia anonimizada de configuración.
- Activar las reglas en modo simulación.
- Comparar coincidencias con una muestra revisada por analistas.
- Corregir condiciones y permisos antes de habilitar confirmación previa.
- Habilitar el modo automático regla por regla, con responsable y fecha de revisión.
- Mantener una opción de desactivación global que detenga nuevos movimientos sin borrar estado ni eventos.

No se requiere una base de datos separada. El respaldo y la recuperación forman parte del procedimiento normal de Odoo y PostgreSQL de la organización.

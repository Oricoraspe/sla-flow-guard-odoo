# SLA Flow Guard para Odoo

**SLA Flow Guard** es un módulo de Odoo diseñado para evitar que un ticket con una dependencia externa válida continúe consumiendo SLA porque el analista olvidó moverlo a la etapa **Falla Externa**.

## Caso de uso

Cuando Odoo asigna automáticamente un ticket, el módulo evalúa reglas configurables basadas en equipo, categoría, servicio, etiqueta, origen y proveedor. Si el caso corresponde inequívocamente a una falla externa, SLA Flow Guard:

1. guarda la etapa actual;
2. mueve el ticket a **Falla Externa**;
3. registra qué regla provocó el movimiento;
4. crea una actividad para que el analista confirme la causa;
5. controla vencimientos y recordatorios;
6. restaura la etapa operativa cuando la dependencia termina o no se confirma.

El módulo no mueve todos los tickets por el simple hecho de haber sido asignados. Si el analista está trabajando activamente y no existe una dependencia externa, el SLA debe continuar. Esta condición protege la calidad de los indicadores y evita pausas indebidas.

## Entregables

- [Definición del producto y caso de uso](docs/actividad.md)
- [Stack tecnológico y estructura del módulo](docs/arquitectura.md)
- [Steering personalizado de Kiro](.kiro/steering/sla-flow-guard.md)

## Resultado esperado

El analista deja de depender de recordar manualmente la etapa correcta, Odoo conserva el cálculo oficial del SLA y la organización obtiene una trazabilidad completa de cada pausa: motivo, regla, responsable, duración y evidencia de la dependencia externa.

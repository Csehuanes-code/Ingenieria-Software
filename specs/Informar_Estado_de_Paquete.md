# Feature Specification: Informar estado de paquete

**Created:** 2026-02-27

------------------------------------------------------------------------

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Sincronización de Estados para Liquidación Financiera (Priority: P1)

Como Módulo de Gestión de Paquetes, quiero centralizar el envío de
estados finales, evidencias y valores comerciales al Módulo de Finanzas
para automatizar el pago de paradas y el cobro de seguros.

**Why this priority:**\
Es el trigger contable del sistema. Sin este informe, el Módulo 3 no
puede diferenciar entre un pago al 100% (Entregado) o una penalidad
(Dañado/Extraviado), bloqueando la liquidación de los transportadores y
el flujo de caja.

------------------------------------------------------------------------

### Independent Test

Simular un cambio de estado a `Entregado` (vía Módulo 2) o `Dañado` (vía
Novedades del Módulo 1) y verificar mediante logs que el Módulo de
Finanzas reciba el JSON con el UUID y el Valor Declarado
correspondiente.

------------------------------------------------------------------------

## Acceptance Scenarios

### Scenario: Notificación de entrega exitosa (Flujo Directo)

**Given** un paquete cuyo estado fue actualizado a `Entregado` por el
Módulo 2.\
**When** el sistema detecta este cambio en el ciclo de vida del
paquete.\
**Then** el sistema informa directamente al Módulo 3 el cumplimiento del
servicio para autorizar el pago del 100% de la tarifa por parada.

------------------------------------------------------------------------

### Scenario: Notificación de incidencia económica (Flujo desde Novedad)

**Given** un paquete que acaba de pasar por el proceso de
`Gestionar novedad de paquete` con estado `Dañado` o `Extraviado`.\
**When** el caso de uso es invocado obligatoriamente por la gestión de
la novedad.\
**Then** el sistema envía al Módulo 3 el estado de falla, el Valor
Declarado y el enlace a la evidencia para aplicar los descuentos o
cobros de póliza correspondientes.

------------------------------------------------------------------------

### Scenario: Notificación de retorno por devolución

**Given** un paquete registrado en el sistema con novedad de
`Devolución`.\
**When** el sistema procesa el re-ingreso a bodega.\
**Then** el sistema informa al Módulo 3 para que liquide el porcentaje
de pago por logística inversa según la matriz de costos de retorno.

------------------------------------------------------------------------

## Edge Cases

**Timeout del Módulo de Finanzas:**\
Si el Módulo 3 no responde, el sistema encola el mensaje en estado
`Pendiente` y realiza reintentos automáticos cada 5 minutos hasta
confirmar la recepción.

**Valor Declarado ausente:**\
Si el paquete no tiene Valor Declarado registrado, el sistema genera una
excepción interna y notifica al administrador antes de permitir el
cierre financiero del UUID.

------------------------------------------------------------------------

## Requirements *(mandatory)*

### Functional Requirements

-   **FR-001:** El sistema DEBE construir un paquete de datos (Payload)
    que incluya: UUID, Estado Final, Valor Declarado e ID del
    Transportador.\
-   **FR-002:** El sistema DEBE adjuntar la referencia o URL de la
    evidencia fotográfica o firma digital (POD) cuando el estado lo
    requiera.\
-   **FR-003:** El sistema DEBE registrar la confirmación (ACK) del
    Módulo 3 para marcar el paquete como `Sincronizado Contablemente`.

------------------------------------------------------------------------

## Key Entities

-   **UUID:** Identificador único que garantiza que Finanzas liquide el
    paquete correcto.\
-   **Valor Declarado:** Monto económico capturado en la admisión que
    define el límite de responsabilidad del seguro.\
-   **Estado Final:** Atributo (`Entregado`, `Dañado`, `Extraviado`,
    `Devolución`) que determina el porcentaje de pago o cobro.

------------------------------------------------------------------------

## Success Criteria *(mandatory)*

### Measurable Outcomes

-   **SC-001:** El 100% de los paquetes que finalicen su ciclo operativo
    deben generar un registro de comunicación exitosa con el Módulo 3.\
-   **SC-002:** El tiempo de latencia entre el cambio de estado y la
    recepción del dato en el Módulo de Finanzas no debe exceder los 10
    segundos en condiciones normales.

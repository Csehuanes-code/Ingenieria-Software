# Feature Specification: Informar Estado de Paquete (MOD1-UC-009)

**Created**: 2026-02-28

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Sincronización de estados finales con el Módulo de Finanzas (Priority: P1)

Como Sistema (Módulo 1), necesito centralizar y enviar al Módulo 3 los estados finales del paquete (Dañado, Extraviado, Devolución) junto con el valor declarado y las evidencias, para que el Módulo de Finanzas pueda ejecutar automáticamente los pagos, descuentos o cobros de póliza que correspondan.

**Why this priority**: Es el trigger contable del sistema. Sin este informe el Módulo 3 no puede distinguir entre una penalidad por daño (Dañado/Extraviado) o una liquidación parcial (Devolución), bloqueando completamente la liquidación de transportadores y el flujo de caja. Es invocado obligatoriamente por [Gestionar Novedad de Paquete (MOD1-UC-008)](./MOD1-UC-008-Gestionar-Novedad-De-Paquete.md).

**Independent Test**: Puede probarse simulando un cambio de estado a `Dañado` , y verificando mediante logs que el Módulo 3 reciba el JSON con el UUID, estado final y valor declarado, y que el sistema registre el ACK de confirmación de recepción.

**Acceptance Scenarios**:

1. **Scenario**: Notificación de incidencia económica por novedad (Dañado o Extraviado)
   - **Given** un paquete que acaba de pasar por [Gestionar Novedad de Paquete (MOD1-UC-008)](./MOD1-UC-008-Gestionar-Novedad-De-Paquete.md) con estado `Dañado` o `Extraviado`.
   - **When** el caso de uso es invocado obligatoriamente por la gestión de la novedad.
   - **Then** el sistema envía al Módulo 3 el payload con estado de falla, valor declarado y URL de la evidencia fotográfica adjunta, para que aplique los descuentos al transportador o ejecute el cobro de la póliza de seguro correspondiente.

2. **Scenario**: Notificación de retorno por devolución
   - **Given** un paquete registrado con novedad de tipo `Devolución` en el Módulo 1.
   - **When** el sistema procesa el re-ingreso a bodega del paquete devuelto.
   - **Then** el sistema informa al Módulo 3 con el estado `Devolución` y el valor declarado, para que liquide el porcentaje de pago correspondiente a la logística inversa según la matriz de costos de retorno definida.

---

### Edge Cases

- What happens when el Módulo 3 no responde a la notificación enviada (timeout)? El sistema encola el mensaje en estado `Pendiente` y ejecuta reintentos automáticos cada 5 minutos hasta confirmar la recepción del ACK. El paquete queda en estado `Pendiente Sincronización Contable` hasta que el Módulo 3 confirme la recepción.
- What happens when el paquete no tiene Valor Declarado registrado al momento de enviar la notificación? El sistema genera una excepción interna, bloquea el envío al Módulo 3 y notifica al administrador del sistema para que resuelva el dato faltante antes de permitir el cierre financiero del UUID.
- What happens when el Módulo 3 responde con error indicando que el UUID ya fue procesado anteriormente? El sistema registra el error en el log de sincronización y no reintenta el envío, marcando el evento como `Duplicado Ignorado` para revisión manual.
- What happens when la URL de la evidencia fotográfica no está disponible al momento de construir el payload? El sistema envía el payload sin la URL de evidencia pero incluye una marca de `Evidencia Pendiente` en el campo correspondiente, generando una alerta para que el Controlador de Novedades suba el archivo a la brevedad.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST construir un payload que incluya como mínimo: UUID del paquete, Estado Final (Dañado, Extraviado, Devolución), Valor Declarado e ID del Transportador.
- **FR-002**: System MUST adjuntar la referencia o URL de la evidencia fotográfica o firma digital (POD) al payload cuando el estado final sea `Dañado`, `Extraviado` o `Entregado`.
- **FR-003**: System MUST registrar la confirmación (ACK) recibida del Módulo 3 y marcar el paquete como `Sincronizado Contablemente` en el historial de estados.

### Key Entities *(include if feature involves data)*

- **UUID**: Identificador único del paquete que garantiza que el Módulo 3 liquide el envío correcto sin ambigüedades.
- **Valor Declarado**: Monto económico capturado en la admisión que define el límite de responsabilidad del seguro y la base de cálculo de pagos y penalidades.
- **Estado Final**: Atributo que determina el porcentaje de pago o cobro aplicable. Valores posibles: `Dañado`, `Extraviado`, `Devolución`.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los paquetes que finalicen su ciclo operativo (Dañado, Extraviado o Devolución) deben generar un registro de comunicación exitosa con el Módulo 3, confirmado por ACK.
- **SC-002**: El tiempo de latencia entre el cambio de estado en el Módulo 1 y la recepción confirmada del dato en el Módulo 3 no debe exceder los 10 segundos bajo condiciones normales de red.

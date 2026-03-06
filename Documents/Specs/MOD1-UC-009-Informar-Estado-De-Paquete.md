# Feature Specification: Informar Estado de Paquete (MOD1-UC-009)

**Created**: 2026-02-28

## User Scenarios & Testing

### User Story 1 — Sincronización del estado final con el Módulo de Finanzas (P1)

Como Sistema, necesito enviar al `Módulo de Gestión de Finanzas` el estado final del paquete junto con el valor declarado y las evidencias, para que ejecute automáticamente los pagos, descuentos o cobros de póliza correspondientes.

**Why this priority**: Es el cierre contable del ciclo. Sin este informe el `Módulo de Gestión de Finanzas` no puede distinguir entre una entrega exitosa, un daño o una devolución, bloqueando la liquidación de transportadores y el flujo de caja. Es invocado por [MOD1-UC-008](./MOD1-UC-008-Gestionar-Novedad-De-Paquete.md) y también de forma automática al recibir el estado `Entregado` desde el `Módulo de Gestión de Rutas`.

**Independent Test**: Simular un cambio a `Entregado` o `Dañado` y verificar en logs que el `Módulo de Gestión de Finanzas` recibió el payload con UUID, estado final y valor declarado, y que el sistema registró el ACK de confirmación.

**Acceptance Scenarios**:

1. **Entrega exitosa**
   - **Given** el `Módulo de Gestión de Rutas` actualiza el paquete a `Entregado`.
   - **When** el sistema detecta el cambio.
   - **Then** construye el payload con UUID, estado, valor declarado e identificador del transportador, lo envía al `Módulo de Gestión de Finanzas` y registra el ACK.

2. **Novedad económica (Dañado o Extraviado)**
   - **Given** se registró una novedad de tipo `Dañado` o `Extraviado` en [MOD1-UC-008](./MOD1-UC-008-Gestionar-Novedad-De-Paquete.md).
   - **When** este caso de uso es invocado por la gestión de novedad.
   - **Then** envía al `Módulo de Gestión de Finanzas` el payload con estado, valor declarado y referencia a la evidencia, para que aplique descuentos al transportador o ejecute el cobro de la póliza.

3. **Devolución**
   - **Given** se registró una novedad de tipo `Devolución`.
   - **When** este caso de uso es invocado.
   - **Then** informa al `Módulo de Gestión de Finanzas` con el estado y valor declarado, para liquidar el porcentaje correspondiente a logística inversa.

### Edge Cases

- **M3 no responde**: el sistema encola el mensaje y reintenta cada 5 minutos. El paquete queda en `Pendiente Sincronización Contable` hasta recibir ACK.
- **Valor declarado ausente**: el sistema bloquea el envío y notifica al Administrador para resolver el dato antes de permitir el cierre financiero.
- **UUID ya procesado por M3**: el sistema registra el evento como duplicado ignorado y no reintenta.
- **Evidencia no disponible al construir el payload**: el sistema envía el payload con marca de `Evidencia Pendiente` y genera una alerta al Controlador de Novedades.

---

## Requirements

### Functional Requirements

- **FR-001**: Construir el payload con: UUID del paquete, estado final (Entregado / Dañado / Extraviado / Devolución), valor declarado e identificador del transportador.
- **FR-002**: Adjuntar la referencia a la evidencia o firma digital (POD) cuando el estado sea `Dañado`, `Extraviado` o `Entregado`.
- **FR-003**: Registrar el ACK recibido del `Módulo de Gestión de Finanzas` y marcar el paquete como `Sincronizado Contablemente`.

### Key Entities

- **Estado Final**: determina la acción financiera. `Entregado` → pago completo. `Dañado` → penalidad y cobro de póliza. `Extraviado` → indemnización por valor declarado. `Devolución` → pago por logística inversa.
- **Valor Declarado**: monto capturado en la admisión que define el límite de responsabilidad y la base de cálculo de pagos y penalidades.

---

## Success Criteria

- **SC-001**: El 100% de los paquetes con estado final generan un registro de comunicación exitosa con el `Módulo de Gestión de Finanzas`, confirmado por ACK.
- **SC-002**: La latencia entre el cambio de estado y la recepción confirmada por M3 no supera los 10 segundos bajo condiciones normales de red.
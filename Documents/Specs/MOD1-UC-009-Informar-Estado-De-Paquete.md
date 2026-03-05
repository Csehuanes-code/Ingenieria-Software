# Feature Specification: Informar Estado de Paquete (MOD1-UC-009)

**Created**: 2026-02-28

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Sincronización de estados finales con el Módulo de Gestión de Finanzas (Priority: P1)

Como Sistema, necesito construir y enviar al `Módulo de Gestión de Finanzas` el payload con el estado final del paquete (Entregado, Dañado, Extraviado, Devolución), el `valor_declarado` y el `id_transportador` —recuperado del Registro de Carga— para que el `Módulo de Gestión de Finanzas` ejecute automáticamente los pagos, descuentos o cobros de póliza que correspondan.

**Why this priority**: Es el trigger contable del sistema. Sin este informe el `Módulo de Gestión de Finanzas` no puede distinguir entre pago al 100% (`Entregado`), penalidad por daño (`Dañado` / `Extraviado`) o liquidación parcial (`Devolución`), bloqueando la liquidación de transportadores y el flujo de caja. Es invocado obligatoriamente por [Gestionar Novedad de Paquete (MOD1-UC-008)](./MOD1-UC-008-Gestionar-Novedad-De-Paquete.md) ante cualquier novedad, y también por el sistema al recibir el estado `Entregado` desde el `Módulo de Gestión de Rutas`.

**Independent Test**: Puede probarse simulando un cambio de estado a `Entregado` (recibido de M2) o `Dañado` (desde Novedades), y verificando mediante logs que: (1) el payload incluye `uuid`, `estado_final`, `valor_declarado`, `id_transportador` y `url_evidencia` cuando aplica; (2) el `id_transportador` proviene del Registro de Carga asociado al UUID; y (3) el `Módulo de Gestión de Finanzas` respondió con ACK y el paquete quedó como `Sincronizado Contablemente`.

**Acceptance Scenarios**:

1. **Scenario**: Notificación de entrega exitosa al `Módulo de Gestión de Finanzas`
   - **Given** un paquete cuyo estado fue actualizado a `Entregado` por el `Módulo de Gestión de Rutas`, con Registro de Carga asociado que contiene `id_transportador` válido.
   - **When** el sistema detecta el estado `Entregado`.
   - **Then** el sistema recupera `id_transportador` del Registro de Carga, construye el payload con `uuid`, `estado_final = Entregado`, `valor_declarado`, `id_transportador` y `url_pod` (firma digital de entrega), y lo envía al `Módulo de Gestión de Finanzas` para autorizar el pago del 100% de la tarifa. Registra el ACK de confirmación.

2. **Scenario**: Notificación de incidencia económica (Dañado o Extraviado)
   - **Given** un paquete que completó MOD1-UC-008 con tipo `Dañado` o `Extraviado`, con Registro de Carga con `id_transportador`.
   - **When** este caso de uso es invocado por MOD1-UC-008.
   - **Then** el sistema recupera `id_transportador` del Registro de Carga, construye el payload con `uuid`, `estado_final`, `valor_declarado`, `id_transportador` y `url_evidencia` (URL pre-firmada de evidencia), y lo envía al `Módulo de Gestión de Finanzas` para aplicar descuentos o ejecutar cobro de póliza.

3. **Scenario**: Notificación de devolución
   - **Given** un paquete con novedad de tipo `Devolución` registrada en MOD1-UC-008.
   - **When** este caso de uso es invocado por MOD1-UC-008.
   - **Then** el sistema construye el payload con `uuid`, `estado_final = Devolución`, `valor_declarado` e `id_transportador` (del Registro de Carga; `null` si no existe Registro de Carga con campo `origen_novedad = Pre-despacho`), y lo envía al `Módulo de Gestión de Finanzas` para liquidar el porcentaje de logística inversa.

---

### Edge Cases

- What happens when el `Módulo de Gestión de Finanzas` no responde? El sistema encola el mensaje y ejecuta reintentos automáticos cada 5 minutos hasta confirmar ACK. El paquete queda en estado `Pendiente Sincronización Contable`.
- What happens when el paquete no tiene `valor_declarado`? El sistema genera una excepción interna, bloquea el envío y notifica al Administrador del Sistema. El paquete no avanza al cierre financiero hasta que se registre el `valor_declarado`.
- What happens cuando no existe Registro de Carga y por tanto no hay `id_transportador`? Esto ocurre en novedades pre-despacho (daño o extravío en bodega antes de la carga al vehículo). El sistema envía el payload con `id_transportador = null` y el campo `origen_novedad = "Pre-despacho"` para que M3 aplique las reglas de responsabilidad del `Módulo de Gestión de Paquetes`.
- What happens cuando M3 responde que el UUID ya fue procesado? El sistema registra el evento como `Duplicado Ignorado` en el log de sincronización y no reintenta.
- What happens cuando la `url_evidencia` no está disponible al construir el payload? El sistema envía el payload sin URL e incluye `evidencia_estado = "Pendiente"`, generando alerta al Controlador de Novedades para subir el archivo. El cierre financiero no se bloquea por este motivo.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST construir el payload con los campos mínimos: `uuid`, `estado_final` (`Entregado` | `Dañado` | `Extraviado` | `Devolución`), `valor_declarado` (obligatorio, numérico > 0), `id_transportador` (del Registro de Carga; `null` con `origen_novedad = "Pre-despacho"` si no existe).
- **FR-002**: System MUST adjuntar `url_evidencia` (URL pre-firmada) para estados `Dañado` o `Extraviado`, y `url_pod` (firma digital de entrega) para `Entregado`. Si la URL no está disponible, incluir `evidencia_estado = "Pendiente"` y notificar al Controlador de Novedades.
- **FR-003**: System MUST registrar el ACK del `Módulo de Gestión de Finanzas` y marcar el paquete como `Sincronizado Contablemente` en el Historial de Estados.
- **NFR-001**: System MUST completar el envío del payload y recibir el ACK en menos de 10 segundos bajo condiciones de red normales (RTT < 200 ms).

### Key Entities *(include if feature involves data)*

- **Paquete**: Fuente de `uuid` y `valor_declarado` (capturado obligatoriamente en MOD1-UC-001).
- **Registro de Carga**: Fuente del `id_transportador`. Creado en MOD1-UC-007. Es la **única fuente autorizada** del `id_transportador` en MOD1.
- **Estado Final y lógica financiera**: `Entregado` → pago del 100% al transportador; `Dañado` → penalidad al transportador + cobro de póliza; `Extraviado` → indemnización por `valor_declarado`; `Devolución` → pago parcial por logística inversa según matriz de costos.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los paquetes que finalicen su ciclo operativo deben generar comunicación exitosa con M3, confirmada por ACK.
- **SC-002**: La latencia entre el cambio de estado y el ACK de M3 no debe exceder 10 segundos bajo condiciones normales de red.
# Feature Specification: Actualizar Estado de Disponibilidad (MOD1-UC-005)

**Created**: 2026-02-28

## User Scenarios & Testing

### User Story 1 — Confirmar que el paquete está listo para ser cargado (P2)

Como Almacenista, necesito confirmar que un paquete ya fue embalado y está listo en el andén, para que el Coordinador de Despacho pueda proceder con la carga al vehículo.

**Why this priority**: Es el nexo entre bodega y despacho. Al confirmar la disponibilidad, el sistema notifica al `Módulo de Gestión de Rutas` que el paquete está listo, sin re-emitir la solicitud de ruta (que ya ocurrió en la admisión).

**Independent Test**: Seleccionar un paquete en `Clasificado` sin inconsistencias bloqueantes, confirmar la disponibilidad y verificar que el historial registre la transición con fecha/hora e identificador del almacenista, y que se envíe la notificación `paquete_listo` al `Módulo de Gestión de Rutas`.

**Acceptance Scenarios**:

1. **Cambio de estado exitoso**
   - **Given** el paquete está en `Clasificado`, con todos sus datos completos y sin inconsistencias.
   - **When** el almacenista escanea el UUID y confirma la disponibilidad.
   - **Then** el sistema registra la transición con fecha/hora e identificador del almacenista, cambia el estado a `Listo para Despacho` y notifica al `Módulo de Gestión de Rutas` con el evento `paquete_listo`.

2. **Transición inválida bloqueada**
   - **Given** el paquete no está en `Clasificado` (por ejemplo, está en `Recibido en Sede`).
   - **When** el almacenista intenta confirmar disponibilidad.
   - **Then** el sistema bloquea la operación con un mensaje descriptivo indicando el estado actual y los pasos previos requeridos.

3. **Daño detectado durante inspección visual**
   - **Given** el almacenista detecta un daño físico al revisar el paquete.
   - **When** no puede confirmarse la disponibilidad.
   - **Then** el almacenista debe registrar la incidencia a través de [MOD1-UC-008](./MOD1-UC-008-Gestionar-Novedad-De-Paquete.md). El estado no avanza a `Listo para Despacho`.

### Edge Cases

- **Fuera de Tolerancia pendiente**: el sistema bloquea la transición hasta que el Supervisor resuelva la inconsistencia.
- **Dos almacenistas simultáneos**: el sistema da prioridad a la primera operación en cola (FIFO). La segunda recibe un mensaje indicando que el paquete ya fue actualizado, con nombre del responsable y timestamp.
- **M2 no responde a la notificación**: el sistema reintenta hasta 3 veces con intervalos de 30 s. Si persiste, el paquete queda en `Listo para Despacho` y se genera una alerta al Supervisor de Bodega.
- **Paquete ya en `Listo para Despacho` o posterior**: el sistema bloquea cualquier intento de actualización sobre ese estado.

---

## Requirements

### Functional Requirements

- **FR-001**: Permitir la actualización únicamente a usuarios con rol Almacenista, Supervisor de Bodega o Administrador del Sistema.
- **FR-002**: Validar que la transición sea coherente con el ciclo de vida. Solo se permite `Clasificado` → `Listo para Despacho`.
- **FR-003**: Registrar automáticamente fecha/hora (UTC) de cada actualización en el historial.
- **FR-004**: Asociar el identificador del usuario responsable a cada cambio de estado en el historial.
- **FR-005**: Mantener un historial cronológico inmutable de todas las transiciones por paquete.
- **FR-006**: Notificar automáticamente al `Módulo de Gestión de Rutas` con el evento `paquete_listo` al cambiar a `Listo para Despacho`. Esta notificación no re-emite la solicitud de ruta.

### Key Entities

- **Historial de Estados**: registro inmutable por paquete con estado anterior, estado nuevo, fecha/hora y usuario responsable.

---

## Success Criteria

- **SC-001**: El 100% de las actualizaciones válidas quedan en el historial con fecha/hora e identificador del responsable.
- **SC-002**: El 100% de las transiciones inválidas son bloqueadas con mensaje descriptivo.
- **SC-003**: El 100% de los cambios a `Listo para Despacho` envían la notificación `paquete_listo` al `Módulo de Gestión de Rutas`.
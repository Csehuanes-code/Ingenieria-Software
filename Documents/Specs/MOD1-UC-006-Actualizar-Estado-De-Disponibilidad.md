# Feature Specification: Actualizar Estado de Paquete por Novedad (MOD1-UC-006)

**Created**: 2026-02-28

## User Scenarios & Testing

### User Story 6 — Actualización de estado por novedad en bodega (P2)

Como Almacenista, necesito modificar el estado de los paquetes cuando detecte que están dañados o extraviados en la bodega, para mantener la trazabilidad actualizada y alertar al Controlador de Novedades.

**Why this priority**: Es crucial para mantener la integridad de la información del sistema cuando se detectan problemas físicos en bodega (caída, rotura, abolladura, temperatura). Esta actualización genera notificaciones automáticas al Controlador de Novedades para que gestione el caso según [Gestionar Novedad de Paquete (MOD1-UC-007)](./MOD1-UC-007-Gestionar-Novedad-De-Paquete.md).

**Independent Test**: Seleccionar un paquete en bodega con daño visible, registrar el cambio a estado `Novedad en Bodega` con subtipo `Dañado`, adjuntar evidencia fotográfica y verificar que el sistema notifique al Controlador de Novedades y registre la transición en el historial con fecha/hora e identificador del almacenista.

**Acceptance Scenarios**:

1. **Registro de paquete dañado en bodega**
   - **Given** el almacenista detecta un paquete con daños físicos visibles durante inspección en bodega.
   - **When** el almacenista selecciona el paquete por UUID, selecciona el estado `Novedad en Bodega` con subtipo `Dañado` y adjunta evidencia fotográfica.
   - **Then** el sistema registra la transición con fecha/hora e identificador del almacenista, actualiza el estado a `Novedad en Bodega - Dañado`, adjunta la evidencia y notifica automáticamente al Controlador de Novedades para que gestione el caso.

2. **Registro de paquete extraviado en bodega**
   - **Given** el almacenista no encuentra físicamente un paquete que está registrado en el sistema.
   - **When** el almacenista busca el paquete por UUID y confirma que está extraviado.
   - **Then** el sistema actualiza el estado a `Novedad en Bodega - Extraviado`, registra la transición en el historial y notifica al Controlador de Novedades para activar el protocolo de seguro.

### Edge Cases

- **¿Qué ocurre si se intenta guardar una novedad de tipo `Dañado` sin adjuntar evidencia fotográfica o de video?** El sistema bloquea el guardado hasta que se adjunte al menos un archivo multimedia válido (foto o video), dado que la evidencia es obligatoria para este tipo de novedad.
- **¿Cómo maneja el sistema el intento de dos almacenistas de actualizar el estado del mismo paquete al mismo tiempo?** El sistema da prioridad a la primera operación en cola (FIFO). La segunda operación recibe un mensaje indicando que el paquete ya fue actualizado, con el nombre del responsable y el timestamp de la operación.
- **¿Qué sucede si el almacenista intenta registrar una novedad en un paquete que ya está en estado `En Tránsito` o en estados posteriores?** El sistema bloquea cualquier actualización de estado desde bodega.

---

## Requirements

### Functional Requirements

- **FR-001**: Registrar automáticamente fecha/hora (UTC) de cada actualización en el historial.
- **FR-002**: Asociar el identificador del usuario responsable a cada cambio de estado en el historial.
- **FR-003**: Mantener un historial cronológico inmutable de todas las transiciones por paquete.
- **FR-004**: Requerir evidencia fotográfica o video obligatoria para cambios a `Novedad en Bodega - Dañado`.
- **FR-005**: Notificar automáticamente al Controlador de Novedades cuando se registre cualquier estado `Novedad en Bodega` para que gestione el caso según [Gestionar Novedad de Paquete (MOD1-UC-007)](./MOD1-UC-007-Gestionar-Novedad-De-Paquete.md).
- **FR-006**: Bloquear actualizaciones de estado si el paquete ya está en `En Tránsito`, `En Parada de Entrega`, `Entregado` o estados posteriores.

### Key Entities

- **Historial de Estados**: registro inmutable por paquete con estado anterior, estado nuevo, fecha/hora, usuario responsable, evidencias adjuntas (si aplica) y notas adicionales.
- **Novedad en Bodega**: tipo (Dañado / Extraviado), descripción, evidencias adjuntas, fecha/hora de detección, identificador del Almacenista responsable.

---

## Success Criteria

- **SC-001**: El 100% de las actualizaciones válidas quedan en el historial con fecha/hora e identificador del responsable.
- **SC-002**: El 100% de las novedades de tipo `Dañado` tienen evidencia multimedia adjunta antes de guardarse.
- **SC-003**: El 100% de los cambios a estados de `Novedad en Bodega` envían notificación automática al Controlador de Novedades.
- **SC-004**: El 100% de las transiciones inválidas son bloqueadas con mensaje descriptivo indicando el motivo.
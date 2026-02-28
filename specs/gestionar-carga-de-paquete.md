# Feature Specification: Gestionar Carga de Paquete (MOD1-UC-006)

**Created**: 2026-02-28

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Verificación y registro de salida física del paquete (Priority: P2)

Como Coordinador de Despacho, necesito verificar y registrar la carga física de los paquetes que ya tienen una ruta asignada y están en estado `Listo para Despacho`, para confirmar formalmente su salida de la sede y notificar al Módulo 2 que el vehículo está listo para iniciar su recorrido.

**Why this priority**: Es el punto formal de salida física de la sede y el último control del Módulo 1 sobre el paquete antes de que el Módulo 2 tome la custodia en campo. Sin esta operación el sistema no garantiza la continuidad de la cadena de custodia ni puede emitir el estado `En Tránsito`.

**Independent Test**: Puede probarse seleccionando un paquete en estado `Listo para Despacho` con ruta asignada por el Módulo 2, confirmando la carga, y verificando que el sistema registre la operación con fecha, hora y responsable, y que se genere la notificación al Módulo 2 de confirmación de carga física completada.

**Acceptance Scenarios**:

1. **Scenario**: Carga exitosa y notificación al Módulo 2
   - **Given** el paquete tiene estado `Listo para Despacho` y el Coordinador de Despacho está autenticado.
   - **When** el coordinador escanea el UUID del paquete y confirma su carga al vehículo.
   - **Then** el sistema registra la operación de carga con fecha, hora e ID del coordinador, y notifica al Módulo 2 que el paquete fue cargado físicamente para que este proceda a confirmar el despacho y emitir el estado `En Tránsito`.

2. **Scenario**: Bloqueo por estado no permitido
   - **Given** el paquete tiene un estado diferente a `Listo para Despacho` (por ejemplo `En Clasificación` o `Fuera de Tolerancia`).
   - **When** el coordinador intenta gestionar la carga del paquete.
   - **Then** el sistema bloquea la operación y muestra un mensaje claro indicando el estado actual del paquete y por qué no permite la carga.

3. **Scenario**: Registro del responsable de la operación
   - **Given** el coordinador de despacho está autenticado en el sistema.
   - **When** confirma la carga del paquete.
   - **Then** el sistema registra el identificador del coordinador asociado al UUID del paquete en el registro de carga, garantizando la trazabilidad de la operación.

---

### Edge Cases

- What happens when dos coordinadores intentan gestionar la carga del mismo paquete simultáneamente? Solo se registra la primera confirmación. La segunda operación es bloqueada con un mensaje indicando que el paquete ya fue gestionado por otro coordinador, mostrando el nombre del responsable.
- What happens when la sesión del usuario expira justo antes de confirmar la carga? El sistema redirige al coordinador a la pantalla de autenticación. No se registra ninguna operación parcial. El paquete permanece en estado `Listo para Despacho` para ser gestionado nuevamente.
- How does system handle un paquete marcado previamente como `Dañado` o `Extraviado` que aparece en el listado de carga? La operación es bloqueada y el sistema muestra un mensaje indicando que el estado del paquete no permite la gestión de carga. El coordinador debe escalar al Controlador de Novedades.
- What happens when el Módulo 2 no responde a la notificación de carga completada? El sistema registra la operación de carga como completada en Módulo 1 y encola la notificación para reintento automático. Genera una alerta visible para el coordinador indicando que la confirmación con el Módulo 2 está pendiente.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST permitir gestionar la carga únicamente de paquetes en estado `Listo para Despacho`.
- **FR-002**: System MUST registrar formalmente la operación de carga del paquete en el historial de trazabilidad.
- **FR-003**: System MUST registrar automáticamente la fecha y hora (timestamp UTC) de la gestión de carga.
- **FR-004**: System MUST asociar el identificador del Coordinador de Despacho responsable a cada operación de carga registrada.
- **FR-005**: System MUST impedir la gestión de carga duplicada sobre un mismo UUID (control de concurrencia).
- **FR-006**: System MUST notificar al Módulo 2 que el paquete ha sido cargado físicamente en el vehículo, para que este proceda a confirmar el despacho y emitir el estado `En Tránsito`.

### Key Entities *(include if feature involves data)*

- **Paquete**: Entidad que contiene UUID, estado del ciclo de vida y atributos logísticos del envío.
- **Coordinador de Despacho**: Usuario responsable de ejecutar y confirmar la operación de carga física en el andén.
- **Registro de Carga**: Entidad que almacena el UUID del paquete, fecha, hora, ID del coordinador responsable y estado de la notificación al Módulo 2.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los paquetes gestionados deben quedar correctamente registrados como operación de carga cuando estén en estado `Listo para Despacho`.
- **SC-002**: El sistema debe bloquear el 100% de los intentos de gestión de carga sobre paquetes en estados no permitidos.
- **SC-003**: El tiempo promedio para gestionar la carga de un paquete (escaneo más confirmación) no debe superar los 3 minutos en el percentil 95.
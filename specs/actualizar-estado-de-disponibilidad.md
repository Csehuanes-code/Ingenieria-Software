# Feature Specification: Actualizar Estado de Disponibilidad (MOD1-UC-005)

**Created**: 2026-02-28

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Confirmación interna de disponibilidad física en bodega (Priority: P2)

Como Almacenista, necesito registrar formalmente que un paquete ya fue embalado y está físicamente disponible en el andén de carga, para avisarle al Coordinador de Despacho que puede proceder con la gestión de carga del vehículo.

**Why this priority**: Es el nexo operativo interno entre el procesamiento de bodega y el despacho físico. Garantiza que el Coordinador de Despacho solo vea y gestione paquetes que están físicamente listos, evitando intentos de carga sobre paquetes aún en clasificación. Este caso de uso ya no se comunica con el Módulo 2 — esa responsabilidad fue trasladada a `Solicitar Ruta de Paquete`.

**Independent Test**: Puede probarse seleccionando un paquete en estado `En Clasificación` con todos sus datos completos y sin inconsistencias bloqueantes, confirmando el cambio a `Listo para Despacho` y verificando que el historial registre la transición con timestamp e ID del almacenista responsable, sin emitir ningún evento externo al Módulo 2.

**Acceptance Scenarios**:

1. **Scenario**: Cambio de estado exitoso a Listo para Despacho
   - **Given** el paquete está en estado `En Clasificación`, con todos sus datos completos y sin inconsistencias bloqueantes (no está en `Fuera de Tolerancia`).
   - **When** el almacenista escanea el UUID, revisa el resumen de datos y confirma la actualización a `Listo para Despacho`.
   - **Then** el sistema registra el nuevo estado en el historial con timestamp UTC e ID del almacenista, y el paquete queda visible y habilitado para el Coordinador de Despacho en el módulo de gestión de carga.

2. **Scenario**: Bloqueo de transición inválida — paquete no ha pasado por clasificación
   - **Given** el paquete tiene estado `Recibido en Sede` (nunca pasó por el proceso de clasificación de bodega).
   - **When** el almacenista intenta actualizar el estado a `Listo para Despacho`.
   - **Then** el sistema bloquea la transición con un mensaje descriptivo indicando que el paquete debe pasar primero por `Preparar Paquete para Almacenaje` y `Clasificar Paquete por Zona de Destino`.

---

### Edge Cases

- What happens when el paquete tiene estado `Fuera de Tolerancia`? El sistema bloquea la transición y muestra el motivo específico de la tolerancia pendiente. El almacenista debe escalar al supervisor para resolver la inconsistencia antes de poder continuar.
- How does system handle si dos almacenistas intentan actualizar el mismo paquete simultáneamente? El sistema utiliza control de concurrencia optimista: la segunda operación detecta el conflicto y notifica al segundo almacenista que el paquete ya fue actualizado, mostrando el estado actual.
- What happens when el paquete presenta daños físicos detectados durante la inspección visual de disponibilidad? El sistema NO permite la transición a `Listo para Despacho`. El almacenista debe registrar la incidencia a través del flujo de `Gestionar Novedad de Paquete`.
- What happens when el almacenista intenta hacer el cambio sobre un paquete ya en estado `Listo para Despacho` o posterior? El sistema bloquea la operación con un mensaje indicando que el estado actual no admite esta transición y muestra el ciclo de vida esperado.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST permitir la actualización del estado de disponibilidad únicamente a usuarios con rol `Almacenista` o superior.
- **FR-002**: System MUST validar que la transición de estado sea coherente con el ciclo de vida definido, bloqueando transiciones inválidas con mensajes descriptivos. Solo se permite la transición de `En Clasificación` a `Listo para Despacho`.
- **FR-003**: System MUST registrar automáticamente la fecha/hora (timestamp UTC) de la actualización en el historial, sin intervención del almacenista.
- **FR-004**: System MUST asociar el ID del usuario responsable a cada cambio de estado en el historial de transiciones.
- **FR-005**: System MUST mantener un historial cronológico completo e inmutable de todas las transiciones de estado para cada UUID de paquete.
- **NFR-001**: System MUST asegurar que el tiempo promedio de actualización de estado no supere los 2 minutos por paquete.

### Key Entities *(include if feature involves data)*

- **Paquete**: Entidad central cuyo estado operativo interno es actualizado en este flujo.
- **Historial de Estados**: Registro inmutable de transiciones. Cada entrada incluye: UUID_paquete, estado_anterior, estado_nuevo, timestamp UTC e ID_usuario responsable.
- **Almacenista**: Actor que ejecuta y autoriza la actualización operativa interna del estado.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las actualizaciones válidas deben quedar registradas en el historial con timestamp y ID del responsable.
- **SC-002**: El sistema debe impedir el 100% de las transiciones de estado no permitidas según el ciclo de vida definido, mostrando siempre un mensaje descriptivo del motivo del bloqueo.
- **SC-003**: El tiempo promedio de actualización de estado no debe superar los 2 minutos por paquete en el percentil 95.
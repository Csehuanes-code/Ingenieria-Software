# Feature Specification: Actualizar Estado de Disponibilidad (MOD1-UC-005)

**Created**: 2026-02-28

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Confirmación de disponibilidad física para despacho (Priority: P2)

Como Almacenista, necesito registrar formalmente que un paquete completó su clasificación por zona de destino y está físicamente listo y embalado en el andén de carga, para transicionarlo al estado `Listo para Despacho` y habilitarlo en la cola del Coordinador de Despacho.

**Why this priority**: Es la puerta de salida de bodega hacia el andén. Garantiza que el Coordinador de Despacho gestione únicamente paquetes que pasaron por todos los pasos: pesaje, asignación de zona de almacenamiento, clasificación por zona de destino y embalaje final. La solicitud de ruta al `Módulo de Gestión de Rutas` ya fue emitida durante la admisión (MOD1-UC-003); este caso de uso únicamente notifica a M2 que el paquete está físicamente listo para ser cargado, **no re-emite** la solicitud de ruta.

**Independent Test**: Puede probarse seleccionando un paquete en estado `Clasificado` con todos sus datos completos y sin inconsistencias bloqueantes, confirmando el cambio a `Listo para Despacho` y verificando que: (1) el Historial de Estados registre la transición con `timestamp` UTC e `id_usuario`, y (2) el sistema envíe la notificación de disponibilidad al `Módulo de Gestión de Rutas` sin re-emitir el evento `solicitar_ruta`.

**Acceptance Scenarios**:

1. **Scenario**: Cambio de estado exitoso a Listo para Despacho con notificación a M2
   - **Given** el paquete está en estado `Clasificado` (completó [Clasificar Paquete por Zona de Destino (MOD1-UC-006)](./MOD1-UC-006-Clasificar-Paquete-Por-Zona-Destino.md)), con todos sus datos completos y sin inconsistencias bloqueantes.
   - **When** el almacenista escanea el UUID, revisa el resumen de datos y confirma la actualización a `Listo para Despacho`.
   - **Then** el sistema registra el cambio en el Historial de Estados con `timestamp` UTC e `id_usuario`, actualiza el estado del paquete a `Listo para Despacho` y envía al `Módulo de Gestión de Rutas` la notificación `paquete_listo` (distinta del evento `solicitar_ruta`) indicando que el paquete está disponible físicamente para ser asignado a un vehículo en la ruta ya planificada.

2. **Scenario**: Bloqueo por estado incorrecto — paquete no completó clasificación
   - **Given** el paquete tiene estado `En Clasificación` (aún no completó [Clasificar Paquete por Zona de Destino (MOD1-UC-006)](./MOD1-UC-006-Clasificar-Paquete-Por-Zona-Destino.md)) o cualquier estado anterior.
   - **When** el almacenista intenta actualizar el estado a `Listo para Despacho`.
   - **Then** el sistema bloquea la transición con un mensaje descriptivo: `"El paquete debe completar Clasificar por Zona de Destino y alcanzar estado 'Clasificado' antes de poder pasar a 'Listo para Despacho'."` Se muestra el estado actual y el siguiente paso requerido.

---

### Edge Cases

- What happens when el paquete tiene estado `Fuera de Tolerancia`? El sistema bloquea la transición y muestra el subtipo de tolerancia pendiente (`Peso` o `Sin GPS`). El almacenista debe escalar al Supervisor de Bodega para resolver la inconsistencia.
- How does system handle si dos almacenistas intentan actualizar el mismo paquete simultáneamente? El sistema implementa control de concurrencia con prioridad FIFO: la primera operación recibida en la cola es procesada exitosamente. La segunda operación, al intentar actualizar la versión ya modificada de la entidad, detecta el conflicto de versión y recibe el mensaje: `"El paquete ya fue actualizado por [nombre del primer almacenista] a las [timestamp]. Estado actual: Listo para Despacho."` No se permite la segunda actualización.
- What happens when el paquete presenta daños físicos detectados durante la inspección visual previa al despacho? El sistema NO permite la transición a `Listo para Despacho`. El almacenista debe registrar la incidencia en [Gestionar Novedad de Paquete (MOD1-UC-008)](./MOD1-UC-008-Gestionar-Novedad-De-Paquete.md).
- What happens when el almacenista intenta actualizar un paquete ya en estado `Listo para Despacho` o en cualquier estado posterior (`En Carga`, `En Tránsito`, etc.)? El sistema bloquea la operación y muestra: `"La transición solicitada no es válida desde el estado actual '[estado_actual]'. Los estados posteriores a 'Listo para Despacho' no pueden revertirse desde este caso de uso."` Se muestra el ciclo de vida completo como referencia.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST permitir la actualización del estado únicamente a usuarios con los siguientes roles: `Almacenista`, `Supervisor de Bodega` o `Administrador del Sistema`. Los roles de `Coordinador de Despacho` y `Empleado de Envío y Recepción` **no** tienen permiso para ejecutar esta transición.
- **FR-002**: System MUST validar que la transición sea coherente con el ciclo de vida. La **única transición permitida** en este caso de uso es `Clasificado` → `Listo para Despacho`. Cualquier intento de transición desde un estado diferente a `Clasificado` debe ser bloqueado con un mensaje descriptivo que indique el estado actual y el estado requerido.
- **FR-003**: System MUST registrar automáticamente el `timestamp` (UTC) de la actualización en el Historial de Estados, sin intervención del almacenista.
- **FR-004**: System MUST asociar el `id_usuario` del almacenista responsable a cada entrada del Historial de Estados.
- **FR-005**: System MUST mantener el Historial de Estados como un registro cronológico, completo e inmutable para cada UUID.
- **FR-006**: System MUST enviar al `Módulo de Gestión de Rutas` la notificación `paquete_listo` cuando el estado cambie a `Listo para Despacho`. Esta notificación es **distinta** del evento `solicitar_ruta` (ya emitido en MOD1-UC-003 durante la admisión) e informa a M2 que el paquete está físicamente disponible en el andén para ser asignado al vehículo de su ruta planificada. Si M2 no responde, encolar la notificación para reintento automático sin revertir el estado del paquete.
- **NFR-001**: System MUST asegurar que el tiempo promedio de actualización de estado no supere 2 minutos por paquete.

### Key Entities *(include if feature involves data)*

- **Paquete**: Estado debe ser `Clasificado` al momento de esta transición.
- **Historial de Estados**: `id`, `uuid_paquete` (FK), `estado_anterior`, `estado_nuevo`, `timestamp` (UTC), `id_usuario`, `rol_usuario`.
- **Actores con permiso de ejecución**: `Almacenista`, `Supervisor de Bodega`, `Administrador del Sistema`.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las transiciones válidas deben quedar registradas en el Historial de Estados con `timestamp` UTC e `id_usuario`.
- **SC-002**: El sistema debe impedir el 100% de las transiciones desde estados distintos a `Clasificado`, mostrando siempre el mensaje descriptivo de bloqueo.
- **SC-003**: El 100% de las actualizaciones exitosas a `Listo para Despacho` deben enviar la notificación `paquete_listo` al `Módulo de Gestión de Rutas` (o encolar el reintento si M2 no responde).
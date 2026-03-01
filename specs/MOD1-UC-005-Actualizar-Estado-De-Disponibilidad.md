# Feature Specification: Actualizar Estado de Disponibilidad (MOD1-UC-005)

**Created**: 2026-02-28

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Confirmación de disponibilidad física y disparo de solicitud de ruta (Priority: P2)

Como Almacenista, necesito registrar formalmente que un paquete ya fue embalado y está físicamente listo en el andén de carga, para que el sistema emita automáticamente la solicitud de ruta al Módulo 2 y el Coordinador de Despacho pueda proceder con la gestión de carga del vehículo.

**Why this priority**: Es el nexo operativo entre el procesamiento de bodega y la planificación de rutas del Módulo 2. Al cambiar el estado a `Listo para Despacho`, el paquete ya tiene todos sus datos completos (peso, volumen, coordenadas GPS, tipo de mercancía, método de pago, valor declarado), cumpliendo los requisitos del payload definido en el Contrato de Integración.

**Independent Test**: Puede probarse seleccionando un paquete en estado `En Clasificación` con todos sus datos completos y sin inconsistencias bloqueantes, confirmando el cambio a `Listo para Despacho` y verificando que: (1) el historial registre la transición con timestamp e ID del almacenista, y (2) el evento [Solicitar Ruta de Paquete(MOD1-UC-003)](./MOD1-UC-003-Solicitar-Ruta-De-Paquete.md) sea emitido al Módulo 2 con el payload completo según el Contrato de Integración.

**Acceptance Scenarios**:

1. **Scenario**: Cambio de estado exitoso y emisión del evento [Solicitar Ruta de Paquete(MOD1-UC-003)](./MOD1-UC-003-Solicitar-Ruta-De-Paquete.md)
   - **Given** el paquete está en estado `En Clasificación`, con todos sus datos completos (peso, volumen, coordenadas GPS, tipo de mercancía, método de pago, valor declarado) y sin inconsistencias bloqueantes.
   - **When** el almacenista escanea el UUID, revisa el resumen de datos y confirma la actualización a `Listo para Despacho`.
   - **Then** el sistema registra el cambio en el historial con timestamp UTC e ID del almacenista, e invoca automáticamente el caso de uso [Solicitar Ruta de Paquete(MOD1-UC-003)](./MOD1-UC-003-Solicitar-Ruta-De-Paquete.md), que construye y envía el evento [Solicitar Ruta de Paquete(MOD1-UC-003)](./MOD1-UC-003-Solicitar-Ruta-De-Paquete.md) al Módulo 2 con el payload completo.

2. **Scenario**: Bloqueo de transición inválida — paquete no ha pasado por clasificación
   - **Given** el paquete tiene estado `Recibido en Sede` (nunca pasó por el proceso de clasificación de bodega).
   - **When** el almacenista intenta actualizar el estado a `Listo para Despacho`.
   - **Then** el sistema bloquea la transición con un mensaje descriptivo indicando que el paquete debe pasar primero por `Preparar Paquete para Almacenaje` y `Clasificar Paquete por Zona de Destino`.

3. **Scenario**: El Módulo 2 rechaza el evento [Solicitar Ruta de Paquete(MOD1-UC-003)](./MOD1-UC-003-Solicitar-Ruta-De-Paquete.md) — reversión de estado
   - **Given** el almacenista confirmó el cambio a `Listo para Despacho` y el evento fue enviado al Módulo 2.
   - **When** el Módulo 2 responde con error detallando el campo inválido o ausente.
   - **Then** el sistema revierte el estado del paquete a `En Clasificación`, notifica al almacenista con el campo específico que causó el rechazo y habilita la corrección para reintentar.

---

### Edge Cases

- What happens when el Módulo 2 rechaza el evento [Solicitar Ruta de Paquete(MOD1-UC-003)](./MOD1-UC-003-Solicitar-Ruta-De-Paquete.md) por campo inválido? El sistema revierte el estado a `En Clasificación`, notifica al almacenista con el detalle del error y permite corregir el dato para reintentar la transición completa.
- What happens when el paquete tiene estado `Fuera de Tolerancia`? El sistema bloquea la transición y muestra el motivo específico de la tolerancia pendiente. El almacenista debe escalar al supervisor para resolver la inconsistencia antes de poder continuar.
- How does system handle si dos almacenistas intentan actualizar el mismo paquete simultáneamente? El sistema utiliza control de concurrencia optimista: la segunda operación detecta el conflicto y notifica al segundo almacenista que el paquete ya fue actualizado, mostrando el estado actual.
- What happens when el paquete presenta daños físicos detectados durante la inspección visual de disponibilidad? El sistema NO permite la transición a `Listo para Despacho`. El almacenista debe registrar la incidencia a través del flujo de [Gestionar Novedad de Paquete(MOD1-UC-008)](./MOD1-UC-008-Gestionar-Novedad-De-Paquete.md).
- What happens when el Módulo 2 no responde al recibir el evento? El sistema reintenta un máximo de 3 veces con intervalos de 30 segundos. Si el fallo persiste, el paquete permanece en `Listo para Despacho` y se genera una alerta al despachador para gestión manual.
- What happens when el almacenista intenta actualizar un paquete ya en estado `Listo para Despacho` o posterior? El sistema bloquea la operación con un mensaje indicando que el estado actual no admite esta transición y muestra el ciclo de vida esperado.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST permitir la actualización del estado de disponibilidad únicamente a usuarios con rol `Almacenista` o superior.
- **FR-002**: System MUST validar que la transición de estado sea coherente con el ciclo de vida definido, bloqueando transiciones inválidas con mensajes descriptivos. Solo se permite la transición de `En Clasificación` a `Listo para Despacho`.
- **FR-003**: System MUST registrar automáticamente la fecha/hora (timestamp UTC) de la actualización en el historial, sin intervención del almacenista.
- **FR-004**: System MUST asociar el ID del usuario responsable a cada cambio de estado en el historial de transiciones.
- **FR-005**: System MUST mantener un historial cronológico completo e inmutable de todas las transiciones de estado para cada UUID de paquete.
- **FR-006**: System MUST invocar automáticamente el caso de uso [Solicitar Ruta de Paquete(MOD1-UC-003)](./MOD1-UC-003-Solicitar-Ruta-De-Paquete.md) cuando el estado cambie a `Listo para Despacho`, el cual emitirá el evento [Solicitar Ruta de Paquete(MOD1-UC-003)](./MOD1-UC-003-Solicitar-Ruta-De-Paquete.md) al Módulo 2 con todos los campos requeridos en el Contrato de Integración v1.0.
- **FR-007**: System MUST revertir el estado a `En Clasificación` si el Módulo 2 rechaza el evento [Solicitar Ruta de Paquete(MOD1-UC-003)](./MOD1-UC-003-Solicitar-Ruta-De-Paquete.md), notificando al almacenista con el detalle del campo inválido.
- **NFR-001**: System MUST asegurar que el tiempo promedio de actualización de estado no supere los 2 minutos por paquete.
- **NFR-002**: System MUST garantizar que la emisión del evento [Solicitar Ruta de Paquete(MOD1-UC-003)](./MOD1-UC-003-Solicitar-Ruta-De-Paquete.md) se realice de forma transaccional: o se actualiza el estado Y se emite el evento, o ninguna de las dos operaciones se persiste.

### Key Entities *(include if feature involves data)*

- **Paquete**: Entidad central cuyo estado operativo cambia en este flujo. En el momento de la transición debe tener completos: `peso_kg`, `volumen_m3`, `tipo_mercancia`, `latitud`, `longitud`, `metodo_pago` y `valor_declarado`.
- **Historial de Estados**: Registro inmutable de transiciones. Cada entrada incluye: UUID_paquete, estado_anterior, estado_nuevo, timestamp UTC e ID_usuario responsable.
- **Almacenista**: Actor que ejecuta y autoriza la actualización operativa del estado.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las actualizaciones válidas deben quedar registradas en el historial con timestamp y ID del responsable.
- **SC-002**: El sistema debe impedir el 100% de las transiciones de estado no permitidas según el ciclo de vida definido, mostrando siempre un mensaje descriptivo del motivo del bloqueo.
- **SC-003**: El 100% de las actualizaciones exitosas a `Listo para Despacho` deben disparar el evento [Solicitar Ruta de Paquete(MOD1-UC-003)](./MOD1-UC-003-Solicitar-Ruta-De-Paquete.md) al Módulo 2, o revertir el estado a `En Clasificación` si ocurre un fallo.
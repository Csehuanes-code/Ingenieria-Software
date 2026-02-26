# Feature Specification: Actualizar Estado de Disponibilidad (MOD1-UC-005)

**Created**: 2026-02-26

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Emisión de solicitud de ruta (Priority: P2-Alta)

Como Almacenista, necesito registrar formalmente que un paquete está listo para salir de bodega, desencadenando la solicitud de ruta.

**Why this priority**: Es el disparador principal para que el sistema emita el evento 'solicitar_ruta' al Módulo 2, siendo el nexo operativo vital entre el procesamiento interno y la planificación de rutas.

**Independent Test**: Puede probarse cambiando el estado de un paquete válido a 'Listo para Despacho' y confirmando que el evento de integración 'solicitar_ruta' es emitido con todos los atributos físicos y comerciales requeridos.

**Acceptance Scenarios**:

1. **Scenario**: Cambio de estado y emisión del evento
   - **Given**: El paquete está en estado 'En Clasificación' con todos sus datos completos y sin inconsistencias bloqueantes.
   - **When**: El almacenista escanea el UUID, verifica el resumen de datos y confirma la actualización a 'Listo para Despacho'.
   - **Then**: El sistema registra el cambio en el historial con fecha y usuario, y emite automáticamente el evento 'solicitar_ruta' al Módulo 2.

### Edge Cases

- What happens when: el Módulo 2 rechaza el evento de solicitud de ruta? El sistema revierte el estado a 'En Clasificación', notifica al almacenista detallando el campo inválido, y permite corregir el dato para reintentar.
- What happens when: el paquete tiene estado 'Fuera de Tolerancia'? El sistema bloquea la transición mostrando el motivo, y el almacenista debe escalar al supervisor para resolver la inconsistencia.
- How does system handle: si dos almacenistas intentan actualizar el mismo paquete simultáneamente? El sistema utiliza control de concurrencia optimista, notificando al segundo usuario que el paquete ya fue actualizado con su nuevo estado.
- What happens when: el Módulo 2 no responde al recibir el evento? El sistema reintenta un máximo de 3 veces con intervalos de 30 segundos; si falla, el paquete permanece 'Listo para Despacho' y genera alerta para gestión manual.
- What happens when: el paquete es marcado como 'Dañado' durante la inspección visual de disponibilidad? El sistema NO permite la transición y exige gestionar el paquete mediante el flujo de novedades.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST: permitir la actualización de estado únicamente a usuarios con rol 'Almacenista' o superior.
- **FR-002**: System MUST: validar que la transición sea coherente con el ciclo de vida, bloqueando transiciones inválidas con mensajes descriptivos.
- **FR-003**: System MUST: registrar automáticamente la fecha/hora de la actualización en el historial.
- **FR-004**: System MUST: asociar el ID del usuario responsable a cada cambio de estado.
- **FR-005**: System MUST: mantener un historial cronológico completo de todas las transiciones para cada paquete.
- **FR-006**: System MUST: emitir automáticamente el evento 'solicitar_ruta' al Módulo 2 cuando el estado cambie, incluyendo todos los campos definidos en el Contrato de Integración.
- **FR-007**: System MUST: revertir el estado a 'En Clasificación' si el Módulo 2 rechaza la solicitud.
- **NFR-001**: System MUST: asegurar que el tiempo promedio de actualización de estado no supere los 2 minutos por paquete.
- **NFR-002**: System MUST: garantizar que la emisión del evento se realice de forma transaccional (o se actualiza y emite, o ninguna operación se persiste).

### Key Entities *(include if feature involves data)*

- **Paquete**: Entidad central cuyo estado es actualizado en el ciclo de vida.
- **Historial de Estados**: Registro inmutable de transiciones que incluye UUID, estado anterior, estado nuevo, timestamp e ID del usuario.
- **Almacenista**: Actor que ejecuta y autoriza la actualización operativa.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las actualizaciones válidas deben quedar registradas en el historial con timestamp y responsable.
- **SC-002**: El sistema debe impedir el 100% de las transiciones de estado no permitidas según el ciclo de vida.
- **SC-003**: El 100% de las actualizaciones a 'Listo para Despacho' deben disparar exitosamente el evento al Módulo 2 o revertir si ocurre un fallo.
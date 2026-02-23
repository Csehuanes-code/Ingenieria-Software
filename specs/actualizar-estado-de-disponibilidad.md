# Feature Specification: Actualizar estado de disponibilidad

**Created**: 2026-02-23

------------------------------------------------------------------------

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Actualizar estado de disponibilidad (Priority: P2)

Como Almacenista, quiero actualizar el estado de disponibilidad del
paquete para reflejar su condición operativa dentro de la bodega y
permitir su posterior clasificación o despacho.

**Why this priority**: El estado de disponibilidad determina si un
paquete puede avanzar en el flujo logístico. Sin esta actualización, el
sistema puede permitir operaciones indebidas sobre paquetes no aptos
para almacenamiento o despacho.

**Independent Test**: Se puede probar modificando el estado de un
paquete desde "En Clasificación" a "Listo para Despacho" y verificando
que el sistema registre el cambio, la fecha/hora y el responsable de la
operación.

------------------------------------------------------------------------

### Acceptance Scenarios

1.  **Scenario**: Actualización válida de disponibilidad.
    -   **Given** que el paquete se encuentra físicamente en bodega y en
        estado "En Clasificación".\
    -   **When** el almacenista actualiza el estado a "Listo para
        Despacho".\
    -   **Then** el sistema registra el nuevo estado y almacena la fecha
        y hora del cambio.
2.  **Scenario**: Bloqueo de transición inválida.
    -   **Given** que el paquete tiene estado "En Tránsito".\
    -   **When** el almacenista intenta actualizar el estado de
        disponibilidad.\
    -   **Then** el sistema DEBE impedir la modificación y notificar que
        el estado actual no permite cambios de disponibilidad.
3.  **Scenario**: Registro de responsable del cambio.
    -   **Given** que el almacenista está autenticado en el sistema.\
    -   **When** confirma la actualización del estado.\
    -   **Then** el sistema registra el identificador del almacenista
        asociado al UUID del paquete.

------------------------------------------------------------------------

### Edge Cases

-   ¿Qué ocurre si el paquete cambia de estado mientras otro usuario
    intenta actualizarlo simultáneamente?\
-   ¿Qué sucede si el sistema detecta inconsistencias entre el estado
    físico del paquete y el estado registrado?\
-   ¿Cómo maneja el sistema un paquete marcado como "Dañado" durante la
    actualización de disponibilidad?

------------------------------------------------------------------------

## Requirements *(mandatory)*

### Functional Requirements

-   **FR-001**: El sistema DEBE permitir la actualización del estado de
    disponibilidad únicamente a usuarios con rol de Almacenista.\
-   **FR-002**: El sistema DEBE validar que la transición de estado sea
    coherente con el ciclo de vida definido.\
-   **FR-003**: El sistema DEBE registrar automáticamente la fecha y
    hora de la actualización.\
-   **FR-004**: El sistema DEBE asociar el identificador del usuario
    responsable al cambio de estado.\
-   **FR-005**: El sistema DEBE mantener un historial de cambios de
    estado por cada UUID.

------------------------------------------------------------------------

### Key Entities

-   **Paquete**: Entidad que contiene UUID, estado actual y datos
    logísticos.\
-   **Historial de Estados**: Registro cronológico de transiciones de
    estado del paquete.\
-   **Almacenista**: Usuario autorizado para gestionar disponibilidad en
    bodega.

------------------------------------------------------------------------

## Success Criteria *(mandatory)*

### Measurable Outcomes

-   **SC-001**: El 100% de las actualizaciones válidas deben quedar
    registradas con fecha, hora y responsable.\
-   **SC-002**: El sistema debe impedir en el 100% de los casos las
    transiciones de estado no permitidas.\
-   **SC-003**: El tiempo promedio de actualización de estado no debe
    superar los 2 minutos por paquete.

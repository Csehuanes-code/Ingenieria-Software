# Feature Specification: Gestionar carga de paquete

**Created**: 2026-02-23

------------------------------------------------------------------------

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Gestionar carga de paquete (Priority: P2)

Como Despachador de carga, quiero gestionar la carga de paquete para
registrar formalmente la salida de los paquetes y solicitar la asiganción de ruta al modulo
de gestion de rutas, garantizando la continuidad del ciclo de vida del paquete.

**Why this priority**: La carga del paquete constituye el punto formal
de salida física desde la sede. Sin esta operación, el sistema no puede
asegurar la continuidad del ciclo de vida ni la trazabilidad del
paquete.

**Independent Test**: Se puede probar seleccionando un paquete con
estado "Listo para Despacho", confirmando la carga y verificando que el
sistema registre la operacion de carga, registre la fecha y hora y
almacene el responsable de la operación y envie la solicitud 
al modulo de gestion de ruta.

------------------------------------------------------------------------

### Acceptance Scenarios

1.  **Scenario**: Transición válida de carga y solicitud de ruta.
    -   **Given** que el paquete tiene estado "Listo para Despacho".\
    -   **When** el despachador confirma la carga del paquete.\
    -   **Then** el sistema registra la operación de carga, almacena la
         fecha y hora, registra el responsable y envia la solucitud al modulo de gestión de rutas.
    -    **Scenario**: Bloqueo por estado no permitido.
    -   **Given** que el paquete tiene un estado diferente a "Listo para
        Despacho".\
    -   **When** el despachador intenta gestionar la carga del paquete.\
    -   **Then** el sistema DEBE impedir la operación e informar que el
        estado actual no permite la carga.
3.  **Scenario**: Registro de responsable de operación.
    -   **Given** que el despachador está autenticado en el sistema.\
    -   **When** confirma la carga del paquete.\
    -   **Then** el sistema registra el identificador del despachador
        asociado al UUID del paquete.

------------------------------------------------------------------------

### Edge Cases

-   ¿Qué ocurre si dos despachadores intentan gestionar la carga del
    mismo paquete simultáneamente?\
-   ¿Qué sucede si la sesión del usuario expira justo antes de confirmar
    la carga?\
-   ¿Cómo maneja el sistema un paquete marcado previamente como "Dañado"
    o "Extraviado"?

------------------------------------------------------------------------

## Requirements *(mandatory)*

### Functional Requirements

-   **FR-001**: El sistema DEBE permitir gestionar la carga únicamente
    de paquetes en estado "Listo para Despacho".\
-   **FR-002**: El sistema DEBE registrar la operacion de carga del paquete.\  
-   **FR-003**: El sistema DEBE registrar automáticamente la fecha y
    hora de la gestión de carga.\
-   **FR-004**: El sistema DEBE asociar el identificador del Despachador
    de carga responsable a la operación.\
-   **FR-005**: El sistema DEBE impedir la gestión de carga duplicada de
    un mismo UUID.
-   **FR-006**: El sistema DEBE enviar una solicitud de ruta al modulo
    de gestion de rutas una vez confirmada la carga.\  
------------------------------------------------------------------------

### Key Entities

-   **Paquete**: Entidad que contiene UUID, estado del ciclo de vida,
    fecha de ingreso y atributos logísticos.\
-   **Despachador de carga**: Usuario responsable de ejecutar la
    operación de carga.\
-   **Registro de carga**: Entidad que almacena fecha, hora y
    responsable asociado al paquete.

------------------------------------------------------------------------

## Success Criteria *(mandatory)*

### Measurable Outcomes

-   **SC-001**: El 100% de los paquetes gestionados deben registrarse
    correctamente como operacion de carga cuando este en estado
    "Listo para Desapcho".\
-   **SC-002**: El sistema debe bloquear en el 100% de los casos la
    gestión de carga cuando el estado no sea permitido.\
-   **SC-003**: El tiempo promedio para gestionar la carga de un paquete
    no debe superar los 3 minutos.

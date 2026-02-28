# Feature Specification: Gestionar novedad de paquete

**Created:** 2026-02-27

------------------------------------------------------------------------

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Control de Incidencias en Bodega y Ruta (Priority: P1)

Como Controlador de Novedades, quiero registrar formalmente las
anomalías de los paquetes (Daño, Extravío, Devolución) para asegurar que
la hoja de vida del paquete esté actualizada y activar los protocolos de
seguro o logística inversa con el Módulo 3.

**Why this priority:**\
Es la garantía de integridad de la mercancía. Permite que la empresa
tenga soporte legal y visual antes de penalizar a un transportador o
cobrar una póliza de seguro, evitando pérdidas por reclamos no
documentados en la custodia del Módulo 1.

------------------------------------------------------------------------

### Independent Test

Seleccionar un paquete en estado `Listo para Despacho` o `En Tránsito`,
registrar una novedad de tipo `Dañado` con fotografía o `Devolución` por
dirección errada, y verificar que el sistema invoque el caso de uso
`Informar estado de paquete` enviando la información correspondiente al
Módulo 3.

------------------------------------------------------------------------

## Acceptance Scenarios

### Scenario: Registro de avería física con evidencia obligatoria

**Given** un paquete identificado por su UUID que presenta daños
físicos.\
**When** el Controlador selecciona el estado `Dañado` y adjunta la
evidencia fotográfica.\
**Then** el sistema actualiza el estado interno y ejecuta el caso de uso
`Informar estado de paquete`, enviando el UUID, valor declarado y
evidencia al Módulo 3.

------------------------------------------------------------------------

### Scenario: Declaración de extravío en auditoría de bodega

**Given** un paquete que no se encuentra físicamente tras el proceso de
clasificación o carga.\
**When** el Controlador actualiza el estado a `Extraviado`.\
**Then** el sistema dispara una alerta e invoca
`Informar estado de paquete` para iniciar el proceso de indemnización
basado en el valor declarado.

------------------------------------------------------------------------

### Scenario: Procesamiento de devolución por logística inversa

**Given** un paquete retornado por el Módulo 2 debido a causas externas
(ej. dirección errónea, cliente no encontrado).\
**When** el Controlador registra el re-ingreso en bodega con el estado
`Devolución`.\
**Then** el sistema cambia el estado del paquete a `Listo para Despacho`
para su re-proceso y ejecuta el caso de uso `Informar estado de paquete`
para que el Módulo 3 gestione el ajuste correspondiente.

------------------------------------------------------------------------

## Edge Cases

**Registro de daño sin evidencia:**\
Si se intenta registrar una novedad de `Dañado` sin adjuntar evidencia,
el sistema impide el guardado y muestra una alerta indicando que la
evidencia es obligatoria.

**Devolución con daño:**\
Si un paquete devuelto presenta daños, el sistema permite registrar
simultáneamente las condiciones `Devolución` y `Dañado`, afectando tanto
la logística como el proceso de seguro.

------------------------------------------------------------------------

## Requirements *(mandatory)*

### Functional Requirements

-   **FR-001:** El sistema DEBE permitir la carga de archivos multimedia
    vinculados al UUID para soportar estados de `Dañado`.\
-   **FR-002:** El sistema DEBE invocar obligatoriamente el caso de uso
    `Informar estado de paquete` al finalizar cualquier registro de
    novedad.\
-   **FR-003:** El sistema DEBE permitir la reclasificación de zona para
    paquetes en estado `Devolución` para que puedan volver al flujo de
    despacho.

------------------------------------------------------------------------

## Key Entities

-   **Novedad:** Entidad que registra el tipo de incidencia (Daño,
    Extravío, Devolución), descripción técnica y evidencias.\
-   **Log de Estado:** Historial de movimientos del UUID que permite
    visualizar todas las transiciones, incluyendo devoluciones.

------------------------------------------------------------------------

## Success Criteria *(mandatory)*

### Measurable Outcomes

-   **SC-001:** El 100% de las novedades reportadas deben quedar
    vinculadas al UUID y ser visibles en el historial de trazabilidad
    del paquete.\
-   **SC-002:** Las devoluciones procesadas deben reflejarse en el
    inventario de `Listo para Despacho` en menos de 5 minutos tras el
    registro.

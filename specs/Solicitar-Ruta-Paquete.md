# Feature Specification: Solicitar Ruta de Paquete

**Created**: 2026-02-21

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Estimación de Entrega en Recepción (Priority: P2)

Como Empleado de Envío y Recepción, quiero solicitar una ruta preliminar al momento del ingreso para informar al cliente la fecha estimada de salida y entrega.

**Why this priority**: Mejora la experiencia del cliente al proporcionar expectativas claras sobre el tiempo de despacho desde el primer contacto.

**Independent Test**: Tras el registro del paquete, invocar la función de solicitud de ruta y verificar que el sistema devuelva una ventana de tiempo de despacho basada en la zona de destino.

**Acceptance Scenarios**:

1. **Scenario**: Asignación preliminar de ruta.
* **Given** un paquete recién registrado con destino a una zona específica.
* **When** el empleado solicita la ruta.
* **Then** el sistema vincula el paquete a la cola de despacho de esa zona y proyecta la fecha de entrega.


2. **Scenario**: Cambio de prioridad de ruta.
* **Given** un paquete marcado como "Urgente".
* **When** se solicita la ruta.
* **Then** el sistema busca el primer espacio disponible en la flota de esa zona para priorizar su salida.



### Edge Cases

* ¿Qué sucede si no hay rutas activas o programadas para el destino solicitado?
* ¿Cómo se recalcula la fecha de entrega si el paquete se queda en "Novedad en Bodega" justo después del registro?

## Requirements *(mandatory)*

### Functional Requirements

* **FR-001**: El sistema DEBE filtrar las rutas disponibles basadas en la dirección/zona registrada en el Módulo 1.
* **FR-002**: El sistema DEBE calcular el Tiempo de Despacho estimado.
* **FR-003**: El sistema DEBE permitir la asignación preliminar a un vehículo o zona de carga según la disponibilidad del Módulo 2.

### Key Entities

* **Ruta**: Representa el trayecto y la planificación logística asociada al paquete.
* **Zona de Destino**: Clasificación geográfica derivada de las coordenadas GPS.

## Success Criteria *(mandatory)*

### Measurable Outcomes

* **SC-001**: El 95% de las solicitudes de ruta deben devolver una fecha estimada de entrega con un margen de error de +/- 1 día.
* **SC-002**: Reducción de la incertidumbre del cliente mediante la emisión de una fecha de salida en el comprobante de recepción.

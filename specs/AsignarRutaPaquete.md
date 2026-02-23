Asignar Ruta Paquete:
# Feature Specification: Asignar Ruta Paquete

**Created**: 2026-02-23

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Consolidación y Asignación de Flota (Priority: P1)

Como Sistema (Módulo de Gestión de Rutas), quiero asignar un paquete específico a una ruta de distribución y vehículo concretos, para materializar la planificación logística y permitir su despacho físico.

**Why this priority**: Es el paso donde la planificación ("Solicitar ruta") y la resolución de problemas ("Gestionar paquete no entregado") se convierten en acción. Sin esto, los paquetes se quedan en la zona de almacenamiento.

**Independent Test**: Ingresar un conjunto de paquetes clasificados para la misma zona y verificar que el sistema los agrupe y los asigne al manifiesto de ruta del vehículo disponible con capacidad suficiente.

**Acceptance Scenarios**:

1. **Scenario**: Asignación exitosa desde clasificación.
* **Given** un paquete que ya fue clasificado en su zona de almacenamiento por destino.
* **When** el Módulo de gestión de rutas ejecuta el algoritmo de asignación o el despachador lo asigna manualmente.
* **Then** el sistema vincula el UUID del paquete al ID de la ruta/vehículo y cambia su estado a "Asignado a Ruta - Listo para carga".

2. **Scenario**: Reasignación por novedad (Extensión).
* **Given** un paquete que viene del flujo de "Gestionar paquete no entregado" listo para reprogramación.
* **When** el sistema procesa la solicitud de nueva ruta.
* **Then** el sistema DEBE priorizar su asignación en la próxima ruta disponible hacia esa zona, marcándolo como "Reintento".

### Edge Cases

* ¿Qué sucede si el volumen/peso de los paquetes asignados a una ruta supera la capacidad física del vehículo designado por el Módulo?
* ¿Cómo reacciona la asignación si una ruta se cancela abruptamente (ej. avería del vehículo) después de haber asignado los paquetes?

## Requirements *(mandatory)*

### Functional Requirements

* **FR-001**: El sistema DEBE relacionar de N a 1 (muchos a uno) los UUID de los paquetes con un ID de Ruta específico.
* **FR-002**: El sistema DEBE validar que el estado previo del paquete permita su asignación (ej. no asignar paquetes que estén en estado "Retenido" o "Entregado").
* **FR-003**: El sistema DEBE generar un "Manifiesto de Carga" digital que liste todos los paquetes asignados a esa ruta para el conductor.
* **FR-004**: El sistema DEBE recibir los triggers (puntos de extensión) tanto de solicitudes nuevas como de reprogramaciones por novedades.

### Key Entities

* **Manifiesto de Ruta**: Documento/entidad que agrupa los paquetes asignados a un vehículo y conductor específicos en una fecha determinada.
* **Capacidad de Flota**: Validaciones de peso y dimensiones máximas por ruta.

## Success Criteria *(mandatory)*

### Measurable Outcomes

* **SC-001**: El tiempo de procesamiento para asignar masivamente un lote de 100 paquetes a sus respectivas rutas no debe exceder los 5 segundos.
* **SC-002**: El 0% de los paquetes asignados a una ruta deben tener conflictos de zonas geográficas diferentes a la planificada.

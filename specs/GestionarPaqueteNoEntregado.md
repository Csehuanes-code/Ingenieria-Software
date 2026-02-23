Gestionar Paquete No Entregado:
# Feature Specification: Gestionar Paquete No Entregado

**Created**: 2026-02-23

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Gestión de Novedades y Logística Inversa (Priority: P1)

Como Controlador de Novedades y Logística Inversa, quiero registrar el motivo por el cual un paquete no pudo ser entregado y definir su próximo estado (reprogramar o devolver), para evitar que el paquete quede en un "limbo" logístico.

**Why this priority**: Si un paquete falla en su entrega y no se gestiona, se rompe la cadena de custodia, afectando los indicadores de cumplimiento y generando quejas del cliente. Es un proceso core para la logística inversa.

**Independent Test**: Simular un intento de entrega fallido, acceder al módulo de novedades, registrar la causal (ej. "Dirección errada") y verificar que el sistema habilite la opción de extender el caso de uso hacia "Asignar ruta paquete" para un nuevo intento.

**Acceptance Scenarios**:

1. **Scenario**: Reprogramación de entrega por cliente ausente.
* **Given** un paquete con estado "Intento de entrega fallido".
* **When** el controlador registra la novedad "Cliente no se encontraba" y selecciona "Reprogramar".
* **Then** el sistema actualiza el estado a "En bodega - Reprogramado" y habilita la extensión para solicitar una nueva asignación de ruta.

2. **Scenario**: Devolución al remitente por máximo de intentos.
* **Given** un paquete que ha superado el límite de intentos de entrega configurados.
* **When** el controlador procesa el paquete en el módulo.
* **Then** el sistema DEBE forzar el estado a "Devolución a origen" y generar las etiquetas correspondientes para logística inversa.

### Edge Cases

* ¿Qué pasa si el paquete sufrió daños físicos durante el intento de entrega fallido y no puede ser devuelto ni reprogramado en su estado actual?
* ¿Cómo maneja el sistema la novedad si el cliente rechaza pagar el valor contra entrega en la puerta?

## Requirements *(mandatory)*

### Functional Requirements

* **FR-001**: El sistema DEBE proveer un catálogo estandarizado de causales de no entrega (ej. Dirección incorrecta, Cliente ausente, Rechazado).
* **FR-002**: El sistema DEBE llevar un contador de intentos de entrega por cada UUID de paquete.
* **FR-003**: El sistema DEBE permitir la extensión hacia el caso de uso de asignación de rutas cuando se determine que el paquete es apto para un nuevo intento.
* **FR-004**: El sistema DEBE notificar automáticamente al módulo de finanzas si la no entrega afecta un cobro contra entrega pendiente.

### Key Entities

* **Novedad**: Registro histórico que detalla el motivo, fecha, hora y responsable del intento fallido.
* **Estado Logístico**: Atributo del paquete que determina si está bloqueado, en reprogramación o en devolución.

## Success Criteria *(mandatory)*

### Measurable Outcomes

* **SC-001**: El 100% de los paquetes no entregados deben tener una causal justificada y un próximo paso definido en menos de 12 horas desde el reporte del conductor.
* **SC-002**: El sistema debe bloquear la reprogramación automática si se supera el umbral de 3 intentos fallidos.

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

Gestionar Paquete No Entregado:
# Feature Specification: Gestionar Paquete No Entregado

**Created**: 2026-02-23

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Gestión de Novedades y Logística Inversa (Priority: P3)

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



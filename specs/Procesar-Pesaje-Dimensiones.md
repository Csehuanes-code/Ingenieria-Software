# Feature Specification: Procesar Pesaje y Dimensiones

**Created**: 2026-02-21

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Validación Física de Mercancía (Priority: P1)

Como Empleado de Envío y Recepción, quiero capturar con precisión el peso y volumen del paquete para asegurar que el cobro sea correcto y que el vehículo asignado tenga la capacidad necesaria.

**Why this priority**: Es el dato fundamental para el cálculo del costo de envío y la seguridad de la flota; una mala medición afecta la rentabilidad y la integridad de los vehículos.

**Independent Test**: Puede probarse ingresando valores de peso y dimensiones en un formulario y verificando que el sistema calcule el volumen total (m³) y valide si el tipo de mercancía (estándar, frágil, peligroso) es compatible con dichas medidas.

**Acceptance Scenarios**:

1. **Scenario**: Procesamiento exitoso de dimensiones estándar.
* **Given** un paquete de 10kg con medidas 50x50x50 cm.
* **When** el empleado registra los datos físicos.
* **Then** el sistema calcula el volumen y clasifica la carga según el tipo de mercancía seleccionado.


2. **Scenario**: Validación de límites de peso.
* **Given** un paquete que excede los 50kg (límite para servicios exprés).
* **When** se intenta guardar el pesaje.
* **Then** el sistema emite una alerta de "Carga Especial" y solicita confirmación de manejo.



### Edge Cases

* ¿Qué sucede si el paquete tiene una forma irregular que no se ajusta a largo/ancho/alto?
* ¿Cómo maneja el sistema el cambio de unidades de medida (kg a lb) durante el pesaje?

## Requirements *(mandatory)*

### Functional Requirements

* **FR-001**: El sistema DEBE permitir la entrada de Peso en kg o lb.
* **FR-002**: El sistema DEBE calcular automáticamente el volumen del paquete multiplicando largo, ancho y alto.
* **FR-003**: El sistema DEBE validar que los valores físicos sean mayores a cero.
* **FR-004**: El sistema DEBE permitir la selección del tipo de mercancía (Frágil, Peligroso, Estándar) para aplicar reglas de manejo.

### Key Entities

* **Atributos Físicos**: Representa las métricas del paquete (Peso, Volumen, Dimensiones).
* **Tipo de Mercancía**: Categorización que define el trato logístico del paquete.

## Success Criteria *(mandatory)*

### Measurable Outcomes

* **SC-001**: El cálculo del volumen debe realizarse en menos de 100ms tras ingresar las dimensiones.
* **SC-002**: El 100% de los paquetes admitidos deben contar con datos de peso y volumen registrados.

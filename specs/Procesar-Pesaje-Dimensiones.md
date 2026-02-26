# Feature Specification: Procesar Pesaje y Dimensiones (MOD1-UC-002)

**Created**: 2026-02-26

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Captura de atributos físicos (Priority: P1-Crítica)

Como Empleado de Envío y Recepción, necesito capturar con precisión el peso y las dimensiones del paquete para garantizar su correcta tarificación y selección de vehículo.

**Why this priority**: Es el fundamento de la rentabilidad operativa y la integridad de la flota, proveyendo al Módulo 2 los datos necesarios para optimizar la selección del vehículo.

**Independent Test**: Puede ser probado seleccionando un paquete, ingresando medidas, validando que el sistema aplique la fórmula de cálculo de volumen y verifique si se disparan alertas de manejo especial.

**Acceptance Scenarios**:

1. **Scenario**: Pesaje estándar y cálculo volumétrico
   - **Given**: El paquete existe en el sistema con estado 'Recibido en Sede' y la báscula está calibrada.
   - **When**: El empleado ingresa el peso y las dimensiones (largo, ancho y alto en centímetros).
   - **Then**: El sistema calcula automáticamente el volumen, valida que las medidas sean mayores a cero y asocia el tipo de mercancía.

### Edge Cases

- What happens when: el paquete tiene una forma irregular? El empleado activa el modo 'Dimensiones irregulares', ingresa las dimensiones de la caja imaginaria mínima y el sistema añade una nota de alerta para el almacenista.
- What happens when: el peso excede los límites operativos (> 4.500 kg)? El sistema bloquea el registro mostrando un error, y el caso debe ser escalado al supervisor.
- How does system handle: si el empleado ingresa el peso en libras por error? El sistema detecta la unidad, convierte automáticamente a kilogramos antes de persistir, y muestra el valor en kilogramos para confirmación.
- What happens when: la densidad es anómala (el paquete tiene dimensiones que no corresponden al peso)? El sistema compara el peso volumétrico con el físico y, si la diferencia supera el 30%, emite una alerta de 'Densidad atípica' para revisión.
- What happens when: una mercancía 'Peligrosa' se registra con dimensiones que exceden los límites de seguridad? El sistema bloquea el registro, muestra las restricciones máximas y exige autorización del supervisor.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST: permitir el ingreso de peso en kilogramos (kg) o libras (lb), convirtiendo automáticamente a kg antes de persistir.
- **FR-002**: System MUST: calcular automáticamente el volumen del paquete en metros cúbicos (m<sup>3</sup>) a partir de largo x ancho x alto en centímetros.
- **FR-003**: System MUST: validar que el peso y todas las dimensiones sean estrictamente mayores a cero antes de permitir guardar.
- **FR-004**: System MUST: permitir la selección del tipo de mercancía (Frágil, Peligroso, Estándar) y aplicar las validaciones correspondientes.
- **FR-005**: System MUST: emitir una alerta de 'Carga Especial' y solicitar confirmación explícita cuando el peso supera los 50 kg.
- **FR-006**: System MUST: calcular y mostrar el peso volumétrico para su comparación con el peso físico declarado.
- **NFR-001**: System MUST: completar el cálculo del volumen en menos de 100 milisegundos tras ingresar las dimensiones.
- **NFR-002**: System MUST: persistir el peso con precisión de 2 decimales y el volumen con precisión de 4 decimales.

### Key Entities *(include if feature involves data)*

- **Atributos Físicos**: Subentidad del Paquete que incluye peso en kg, largo, ancho, alto en cm y volumen en m<sup>3</sup>.
- **Tipo de Mercancía**: Enumeración (Frágil, Peligroso, Estándar) que define las reglas de manejo, almacenaje y validaciones aplicables.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los paquetes que avanzan a clasificación deben contar con peso y volumen registrados y validados.
- **SC-002**: El cálculo de volumen debe completarse en menos de 100 milisegundos en el percentil 99.
- **SC-003**: El 0% de los paquetes con peso o dimensiones iguales a cero deben poder avanzar al siguiente estado.
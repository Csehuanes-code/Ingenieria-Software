# Feature Specification: Procesar Pesaje y Dimensiones (MOD1-UC-002)

**Created**: 2026-02-28

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Captura de atributos físicos del paquete (Priority: P1)

Como Empleado de Envío y Recepción, necesito capturar con precisión el peso y las dimensiones del paquete para garantizar su correcta tarificación y proveer al `Módulo de Gestión de Rutas` los datos necesarios para la selección óptima del vehículo.

**Why this priority**: Es el fundamento de la rentabilidad operativa y la integridad de la flota. Una medición incorrecta afecta el cálculo de costos y puede comprometer la seguridad de los vehículos. Este caso de uso es invocado obligatoriamente por [Registrar Admisión de Paquete(MOD1-UC-001)](./MOD1-UC-001-Registrar-Admision-De-Paquete.md).

**Independent Test**: Puede probarse ingresando un conjunto de valores de peso y dimensiones en el formulario y verificando que el sistema calcule el volumen correctamente (largo × ancho × alto), valide que todas las medidas sean mayores a cero y dispare las alertas de manejo especial según el tipo de mercancía.

**Acceptance Scenarios**:

1. **Scenario**: Pesaje estándar y cálculo volumétrico exitoso
   - **Given** el paquete existe en el sistema con estado `Recibido en Sede` y la báscula está calibrada y operativa.
   - **When** el empleado ingresa el peso en kg y las dimensiones en centímetros (largo, ancho y alto).
   - **Then** el sistema guarda el peso registrado, calcula automáticamente el volumen en m³, calcula el peso volumétrico, valida que todas las medidas sean mayores a cero y asocia el tipo de mercancía seleccionado al paquete.

2. **Scenario**: Alerta por peso de carga especial
   - **Given** el empleado ingresa un peso superior a 50 kg para un paquete estándar.
   - **When** el sistema detecta que el valor supera el umbral de carga especial.
   - **Then** el sistema emite una alerta de `Carga Especial` y solicita confirmación explícita del empleado antes de permitir continuar con el registro.

---

### Edge Cases

- What happens when el paquete tiene una forma irregular que no se ajusta a largo/ancho/alto estándar? El empleado activa el modo `Dimensiones irregulares`, ingresa las dimensiones de la caja contenedora imaginaria mínima que envuelve el paquete y el sistema añade una nota de `Forma irregular` al registro para alertar al almacenista.
- What happens when el peso supera los límites operativos (> 4.500 kg)? El sistema bloquea el registro con un error de `Peso fuera del rango operativo del sistema` y el caso debe ser escalado al supervisor para gestión manual.
- What happens when la densidad del paquete es anómala (diferencia mayor al 30% entre peso volumétrico y peso físico)? El sistema emite una alerta de `Densidad atípica` para revisión del empleado antes de permitir continuar con el guardado.
- What happens when una mercancía `Peligrosa` se registra con dimensiones que exceden los límites de seguridad configurados? El sistema bloquea el registro, muestra las restricciones de dimensiones y peso máximas para esa categoría, y exige autorización del supervisor para continuar.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST permitir el ingreso de peso en kilogramos (kg).
- **FR-002**: System MUST calcular automáticamente el volumen del paquete en m³ aplicando la fórmula: V = (largo × ancho × alto) / 1.000.000, donde las dimensiones se ingresan en centímetros.
- **FR-003**: System MUST validar que el peso y todas las dimensiones (largo, ancho, alto) sean estrictamente mayores a cero antes de permitir guardar.
- **FR-004**: System MUST permitir la selección del tipo de mercancía (Frágil, Peligroso, Estándar) y aplicar las validaciones de límites de peso y dimensiones correspondientes a cada categoría.
- **FR-005**: System MUST emitir una alerta de `Carga Especial` y solicitar confirmación explícita del empleado cuando el peso supera los 50 kg.
- **FR-006**: System MUST calcular y mostrar el peso volumétrico para su comparación con el peso físico declarado, emitiendo alerta de `Densidad atípica` si la diferencia supera el 30%.

### Key Entities *(include if feature involves data)*

- **Atributos Físicos**: Subentidad del Paquete que incluye peso_kg, largo_cm, ancho_cm, alto_cm y volumen_m3.
- **Tipo de Mercancía**: Enumeración (Frágil | Peligroso | Estándar) que define las reglas de manejo, almacenaje y validaciones aplicables al paquete.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los paquetes que avanzan a clasificación deben contar con peso_kg y volumen_m3 registrados y validados.
- **SC-002**: El 0% de los paquetes con peso o alguna dimensión igual a cero deben poder avanzar al siguiente estado del ciclo de vida.
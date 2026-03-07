# Feature Specification: Procesar Pesaje y Dimensiones (MOD1-UC-002)

**Created**: 2026-02-28

## User Scenarios & Testing

### User Story 1 — Captura de medidas y cálculo del precio (P1)

Como Empleado de Envío y Recepción, necesito capturar el peso y las dimensiones del paquete para calcular el precio de envío y proveer al `Módulo de Gestión de Rutas` los datos físicos que necesita para asignar el vehículo correcto.

**Why this priority**: Una medición incorrecta afecta el precio cobrado y puede comprometer la seguridad de los vehículos. Es invocado obligatoriamente por [MOD1-UC-001](./MOD1-UC-001-Registrar-Admision-De-Paquete.md). El precio calculado se muestra al cliente durante la admisión y queda fijo al confirmar.

**Independent Test**: Ingresar peso y dimensiones, y verificar que el sistema calcule volumen, peso volumétrico y precio correctamente, emita las alertas según el tipo de mercancía y bloquee valores inválidos.

**Acceptance Scenarios**:

1. **Pesaje estándar exitoso**
   - **Given** el paquete existe en el sistema y la báscula está operativa.
   - **When** el empleado ingresa peso, dimensiones y tipo de mercancía.
   - **Then** el sistema calcula volumen, peso volumétrico, peso facturable y precio de envío, y asocia todos los valores al paquete.

2. **Alerta por Carga Especial**
   - **Given** el empleado ingresa peso mayor a 50 kg o volumen mayor a 0.5 m³.
   - **When** el sistema detecta que supera el umbral.
   - **Then** emite alerta de `Carga Especial` y solicita confirmación explícita antes de continuar.

### Categorías de Paquete

| Tipo de Mercancía | Condiciones | Zona requerida |
|---|---|---|
| Estándar | Sin restricciones especiales | Normal |
| Frágil | Contenido susceptible a impacto o vibración (vidrio, cerámica, electrónicos, etc.). Declarado por el remitente. | Delicada |
| Peligroso | Riesgo de incendio, explosión, toxicidad o corrosión. Requiere documentación de seguridad del remitente. | Alto Riesgo |

| Categoría de Carga | Condiciones |
|---|---|
| Normal | Peso ≤ 50 kg y volumen ≤ 0.5 m³ |
| Carga Especial | Peso > 50 kg AND Peso ≤ 70 kg o volumen > 0.5 m³ AND volumen ≤ 0.7 m³ |

### Edge Cases

- **Forma irregular**: el empleado activa `Dimensiones irregulares`. Los campos pasan a representar las dimensiones de la caja contenedora mínima imaginaria. La fórmula de cálculo no varía.
- **Peso > 70 kg**: el sistema bloquea el registro.
- **Densidad atípica** (diferencia > 30% entre peso volumétrico y peso real): alerta para revisión antes de continuar.
- **Mercancía Peligrosa fuera de límites de seguridad**: el sistema bloquea y exige autenticación del Supervisor de Admisión.

---

## Requirements

### Functional Requirements

- **FR-001**: Aceptar el peso en kilogramos en el rango 0.01–70 kg.
- **FR-002**: Calcular el volumen en m³ con la fórmula `V = (largo × ancho × alto) / 1.000.000`, con dimensiones en centímetros.
- **FR-003**: Validar que el peso y todas las dimensiones sean mayores a cero.
- **FR-004**: Aplicar las restricciones de zona y manejo según el tipo de mercancía seleccionado.
- **FR-005**: Emitir alerta `Carga Especial` y pedir confirmación cuando peso > 50 kg AND peso ≤ 70 kg o volumen > 0.5 m³ AND volumen ≤ 0.7 m³.
- **FR-006**: Calcular el peso volumétrico (`volumen_m3 × 250`) y emitir alerta de `Densidad atípica` si la diferencia con el peso real supera el 30%.
- **FR-007**: Calcular y mostrar el precio de envío con la fórmula:
  ```
  Precio = tarifa base de sede
         + (peso facturable × tarifa por kg)
         + (distancia km × tarifa por km)
         + recargo por tipo de mercancía
         + recargo por categoría de carga
  ```
  El **peso facturable** es el mayor entre el peso real y el peso volumétrico. Las tarifas son configurables por el Administrador del Sistema. El precio es inmutable una vez confirmado el registro.
- **FR-008**: Ofrecer el modo `Dimensiones irregulares` como toggle en el formulario, con ilustración de referencia de la caja contenedora mínima.

### Key Entities

- **Atributos físicos del paquete**: peso, dimensiones (largo, ancho, alto), volumen, peso volumétrico, peso facturable, tipo de mercancía, categoría de carga, indicador de forma irregular.
- **Precio de envío**: valor calculado (inmutable al confirmar), distancia estimada, desglose de componentes tarifarios.

---

## Success Criteria

- **SC-001**: El 100% de los paquetes que avanzan tienen peso y volumen registrados y validados.
- **SC-002**: El 0% de los paquetes con peso o dimensión igual a cero pueden avanzar en el ciclo de vida.
- **SC-003**: Los cálculos se completan en menos de 100 ms en el percentil 99.

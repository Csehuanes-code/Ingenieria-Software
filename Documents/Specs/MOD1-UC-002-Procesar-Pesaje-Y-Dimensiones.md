# Feature Specification: Procesar Pesaje y Dimensiones (MOD1-UC-002)

**Created**: 2026-02-28

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Captura de atributos físicos y cálculo del precio de envío (Priority: P1)

Como Empleado de Envío y Recepción, necesito capturar el peso, las dimensiones y el tipo de mercancía del paquete para calcular automáticamente el precio de envío, garantizar la correcta tarificación y proveer al `Módulo de Gestión de Rutas` los datos físicos necesarios para la selección óptima del vehículo.

**Why this priority**: Es el fundamento de la rentabilidad operativa y la integridad de la flota. Una medición incorrecta afecta el precio cobrado al cliente y puede comprometer la seguridad de los vehículos. Este caso de uso es invocado obligatoriamente por [Registrar Admisión de Paquete (MOD1-UC-001)](./MOD1-UC-001-Registrar-Admision-De-Paquete.md). Su resultado (`precio_envio_calculado`) es mostrado al cliente durante la admisión y queda vinculado permanentemente al UUID del paquete.

**Independent Test**: Puede probarse ingresando valores de peso y dimensiones en el formulario y verificando que el sistema calcule `volumen_m3`, `peso_volumetrico_kg` y `precio_envio_calculado` correctamente, que aplique las restricciones de cada categoría, y que dispare las alertas correspondientes según el tipo de mercancía.

**Acceptance Scenarios**:

1. **Scenario**: Pesaje estándar, cálculo volumétrico y precio de envío exitosos
   - **Given** el paquete existe con estado `Recibido en Sede` (o está en proceso de admisión) y la báscula está calibrada y operativa.
   - **When** el empleado ingresa `peso_kg` y las dimensiones `largo_cm`, `ancho_cm`, `alto_cm`, y selecciona el `tipo_mercancia`.
   - **Then** el sistema calcula `volumen_m3 = (largo_cm × ancho_cm × alto_cm) / 1.000.000`, calcula `peso_volumetrico_kg = volumen_m3 × 250`, determina `peso_facturable_kg = max(peso_kg, peso_volumetrico_kg)`, calcula `precio_envio_calculado` según la fórmula de tarifas (ver FR-007), y asocia todos los valores al paquete.

2. **Scenario**: Alerta por peso de Carga Especial
   - **Given** el empleado ingresa un `peso_kg` superior a 50 kg para un paquete de tipo `Estándar`.
   - **When** el sistema detecta que el valor supera el umbral de Carga Especial.
   - **Then** el sistema emite una alerta de `Carga Especial` y solicita confirmación explícita del empleado. Sin esa confirmación, el formulario no avanza. Al confirmar, se aplica el recargo de Carga Especial en el `precio_envio_calculado`.

---

### Definición de Categorías de Paquete

Las categorías no son mutuamente excluyentes respecto a `tipo_mercancia` y `categoria_carga`. Un paquete puede ser simultáneamente `Frágil` y `Carga Especial`, por ejemplo.

#### Tipo de Mercancía

| Categoría | Condiciones para clasificar | Por qué es importante |
|---|---|---|
| `Estándar` | Paquete que no cumple ninguna condición de Frágil ni Peligroso. Sin restricciones especiales de manejo. | Base del cálculo tarifario; permite asignación a cualquier zona de almacenamiento. |
| `Frágil` | Contiene objetos que pueden romperse, deformarse o deteriorarse por impacto, vibración o presión (ej. vidrio, cerámica, electrónicos sin blindaje, instrumentos musicales, arte). Lo declara el remitente al admitir el paquete. | Requiere zona de almacenamiento de categoría `Delicada`, manipulación con precaución y posicionamiento correcto (ej. "Este lado arriba"). Genera recargo tarifario. |
| `Peligroso` | Contiene sustancias o materiales que representan riesgo de incendio, explosión, toxicidad, corrosión o reactividad (ej. baterías de litio, productos químicos, combustibles, materiales radiactivos). Requiere declaración formal del remitente y documentación de seguridad (hoja de datos de seguridad o MSDS). | Requiere zona de almacenamiento `Alto Riesgo`, documentación adicional obligatoria y límites estrictos de peso y dimensiones por vehículo. Genera recargo tarifario elevado. Exige autorización del Supervisor de Admisión ante límites excedidos. |

#### Categoría de Carga

| Categoría | Condiciones para clasificar | Por qué es importante |
|---|---|---|
| `Normal` | `peso_kg` ≤ 50 kg y `volumen_m3` ≤ 0.5 m³. No aplica ninguna restricción especial de equipo. | Flujo estándar sin recargos adicionales por peso o volumen. |
| `Carga Especial` | `peso_kg` > 50 kg **o** `volumen_m3` > 0.5 m³. Requiere equipos de carga (montacargas, grúa, banda) para su manejo seguro. | Implica recargo tarifario por manipulación especial y restringe los vehículos elegibles para el transporte. |

#### Clasificación de Prioridad (campo `prioridad` en Paquete)

| Prioridad | Condiciones para clasificar | Por qué es importante |
|---|---|---|
| `Estándar` | Todos los paquetes por defecto. El tiempo de entrega sigue la planificación normal de rutas. | Base del cálculo de tiempo estimado de entrega. |
| `Urgente` | El remitente paga explícitamente la tarifa de urgencia al momento de la admisión **y** el destino tiene cobertura de entrega express disponible. Un paquete `Urgente` se asigna al primer vehículo disponible en la zona, tiene prioridad sobre los `Estándar` en la cola de carga y se le garantiza despacho en el mismo día hábil si es admitido antes de las 12:00 hrs. | Permite al `Módulo de Gestión de Rutas` priorizar la asignación de vehículo y ruta. Genera recargo tarifario de urgencia. |

---

### Edge Cases

- What happens when el paquete tiene forma irregular? El empleado activa el modo `Dimensiones irregulares` mediante un toggle visible en el formulario. Los campos de dimensión cambian su etiqueta a `largo máx.`, `ancho máx.` y `alto máx.` con una ilustración de la caja contenedora mínima. La fórmula de cálculo de volumen no varía. Se persiste `es_irregular = true` y la nota `Forma irregular — medidas de caja contenedora mínima`.
- What happens when el peso supera 4.500 kg? El sistema bloquea el registro con el error `Peso fuera del rango operativo` y escala el caso al Supervisor de Admisión para gestión manual fuera del sistema estándar.
- What happens when la densidad es anómala (diferencia > 30% entre `peso_volumetrico_kg` y `peso_kg`)? El sistema emite la alerta `Densidad atípica` con ambos valores mostrados, para que el empleado verifique si hubo error de captura antes de continuar.
- What happens when un paquete `Peligroso` excede los límites de peso o dimensiones configurados para esa categoría? El sistema bloquea el registro, muestra los límites máximos configurados y exige la autenticación del Supervisor de Admisión (ingreso de sus credenciales en el mismo formulario) para habilitar el avance.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST permitir el ingreso de `peso_kg` como valor numérico decimal en el rango [0.01, 4500.00] kg.
- **FR-002**: System MUST calcular `volumen_m3 = (largo_cm × ancho_cm × alto_cm) / 1.000.000`. Las dimensiones se ingresan en centímetros como enteros positivos mayores a cero.
- **FR-003**: System MUST validar que `peso_kg`, `largo_cm`, `ancho_cm` y `alto_cm` sean estrictamente mayores a cero. Cualquier valor ≤ 0 bloquea el avance con un mensaje de error por campo.
- **FR-004**: System MUST permitir la selección del `tipo_mercancia` (`Estándar` | `Frágil` | `Peligroso`) y aplicar las validaciones y límites operativos correspondientes a cada categoría según la tabla de la sección anterior. Los límites por categoría son configurables por el Administrador del Sistema.
- **FR-005**: System MUST emitir la alerta `Carga Especial` y solicitar confirmación explícita del empleado cuando `peso_kg` > 50 kg o `volumen_m3` > 0.5 m³, y registrar `categoria_carga = Carga Especial` en los atributos físicos del paquete.
- **FR-006**: System MUST calcular `peso_volumetrico_kg = volumen_m3 × 250` y emitir la alerta `Densidad atípica` si la diferencia relativa entre `peso_volumetrico_kg` y `peso_kg` supera el 30%.
- **FR-007**: System MUST calcular y mostrar el `precio_envio_calculado` usando la siguiente fórmula:

  ```
  precio_envio_calculado =
      (tarifa_base_sede)
    + (peso_facturable_kg × tarifa_por_kg)
    + (distancia_km × tarifa_por_km)
    + recargo_tipo_mercancia
    + recargo_categoria_carga
    + recargo_prioridad
  ```

  Donde:
  - `peso_facturable_kg = max(peso_kg, peso_volumetrico_kg)` — se cobra el mayor entre peso real y peso volumétrico.
  - `distancia_km` — distancia aproximada en línea recta entre coordenadas GPS de la sede y coordenadas del destinatario, calculada por el sistema.
  - `tarifa_base_sede`, `tarifa_por_kg`, `tarifa_por_km` — valores configurables por sede y por el Administrador del Sistema.
  - `recargo_tipo_mercancia` — valor adicional fijo por categoría: `Estándar = 0`, `Frágil = tarifa_recargo_fragil`, `Peligroso = tarifa_recargo_peligroso`.
  - `recargo_categoria_carga` — `Normal = 0`, `Carga Especial = tarifa_recargo_carga_especial`.
  - `recargo_prioridad` — `Estándar = 0`, `Urgente = tarifa_recargo_urgente`.

  El `precio_envio_calculado` es mostrado al cliente en pantalla antes de confirmar el registro y queda persistido en la entidad Paquete como campo inmutable una vez confirmado el registro.

- **FR-008**: System MUST proveer el modo `Dimensiones irregulares` como toggle en el formulario. Al activarlo, los campos cambian su etiqueta a `largo máx.`, `ancho máx.` y `alto máx.` con una ilustración de referencia. La fórmula de cálculo no varía. El sistema persiste `es_irregular = true`.
- **NFR-001**: System MUST completar el cálculo de `volumen_m3`, `peso_volumetrico_kg` y `precio_envio_calculado` en menos de 100 milisegundos tras el ingreso de las dimensiones.
- **NFR-002**: System MUST persistir `peso_kg` con precisión de 2 decimales, `volumen_m3` con 4 decimales y `precio_envio_calculado` con 2 decimales.

### Key Entities *(include if feature involves data)*

- **Atributos Físicos** (subentidad de Paquete): `peso_kg` (decimal 2d), `largo_cm` (entero), `ancho_cm` (entero), `alto_cm` (entero), `volumen_m3` (decimal 4d), `peso_volumetrico_kg` (decimal 2d), `peso_facturable_kg` (decimal 2d), `tipo_mercancia` (`Estándar` | `Frágil` | `Peligroso`), `categoria_carga` (`Normal` | `Carga Especial`), `es_irregular` (boolean, default false).
- **Precio de Envío** (subentidad de Paquete): `precio_envio_calculado` (decimal 2d, inmutable tras confirmación), `distancia_km` (decimal 2d), `desglose_tarifario` (JSON con los componentes del cálculo para trazabilidad).
- **Tipo de Mercancía**: Enumeración con límites operativos por categoría configurables por el Administrador del Sistema.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los paquetes confirmados deben tener `peso_kg`, `volumen_m3` y `precio_envio_calculado` registrados, validados y mayores a cero.
- **SC-002**: El 0% de los paquetes con `peso_kg` o cualquier dimensión ≤ 0 deben poder avanzar al siguiente estado.
- **SC-003**: El cálculo de `volumen_m3`, `peso_volumetrico_kg` y `precio_envio_calculado` debe completarse en menos de 100 ms en el percentil 99.
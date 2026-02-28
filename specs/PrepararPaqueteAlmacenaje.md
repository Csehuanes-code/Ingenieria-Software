# Feature Specification: Preparar Paquete para Almacenaje (MOD1-UC-003)

**Created**: 2026-02-28

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Organización física de paquetes en bodega (Priority: P1)

Como Almacenista, necesito organizar físicamente los paquetes recibidos en las zonas de almacenamiento sugeridas por el sistema, según su destino geográfico y tipo de mercancía, para garantizar que el Módulo 2 pueda realizar la consolidación de carga de forma eficiente.

**Why this priority**: Garantiza la trazabilidad física del paquete dentro de bodega y elimina los puntos ciegos que impiden la consolidación de carga. Sin la asignación de zona no es posible ejecutar `Clasificar Paquete por Zona de Destino`.

**Independent Test**: Puede probarse escaneando el UUID de un paquete en estado `Recibido en Sede` con datos físicos y GPS completos, verificando que el sistema sugiera la zona correcta según coordenadas y tipo de mercancía, y confirmando que al aceptar la sugerencia el estado cambia a `En Clasificación` y se actualiza el contador de ocupación de la zona asignada.

**Acceptance Scenarios**:

1. **Scenario**: Asignación exitosa a zona sugerida
   - **Given** el paquete existe con estado `Recibido en Sede`, tiene coordenadas GPS válidas y datos físicos registrados, y el almacenista está autenticado.
   - **When** el almacenista escanea el UUID y el sistema calcula y sugiere automáticamente la zona de almacenamiento correspondiente al destino.
   - **Then** el almacenista confirma la zona sugerida, el sistema invoca `Clasificar Paquete por Zona de Destino`, registra la zona asignada, actualiza el estado del paquete a `En Clasificación` y actualiza el contador de ocupación (peso y volumen) de la zona.

2. **Scenario**: Alerta de manejo especial para mercancía frágil o peligrosa
   - **Given** el paquete tiene el tipo de mercancía registrado como `Frágil` o `Peligroso`.
   - **When** el almacenista escanea el UUID para iniciar la preparación de almacenaje.
   - **Then** el sistema emite una alerta de manejo especial, sugiere únicamente zonas aptas para esa categoría (`Zona de Manipulación Delicada` o `Zona de Resguardo de Alto Riesgo`) e impide la asignación a zonas no calificadas.

---

### Edge Cases

- What happens when la zona de destino está saturada (capacidad máxima alcanzada)? El sistema detecta la condición, actualiza el estado de la zona a `Saturado`, emite una alerta de `Zona Saturada` y sugiere automáticamente una zona de contingencia disponible. El paquete asignado a la zona alternativa queda marcado con nota de `Desborde de zona`.
- What happens when el peso físico del paquete en bodega difiere significativamente del registrado en admisión? Si la diferencia supera el umbral configurado (10%), el sistema marca el paquete como `Fuera de Tolerancia`, bloqueando cualquier avance en el ciclo de vida hasta que un supervisor resuelva la inconsistencia.
- What happens when el paquete no tiene coordenadas GPS válidas al momento de escanear? El sistema impide la clasificación, marca el paquete como `Fuera de Tolerancia — Sin GPS` y notifica al Empleado de Envío para que resuelva el dato geográfico.
- How does system handle si dos almacenistas intentan procesar el mismo paquete simultáneamente? El sistema aplica bloqueo optimista: la segunda operación detecta el conflicto y notifica al segundo almacenista que el paquete ya fue procesado, mostrando el estado actual actualizado.
- What happens when un paquete marcado como `Peligroso` intenta ser asignado a una zona normal? El sistema bloquea la asignación, muestra un error explícito e impide confirmar hasta que se seleccione una zona apta para mercancía peligrosa.
- How does system handle si el cliente cambia la dirección de destino después de que el paquete ya fue clasificado? El cambio se gestiona como excepción supervisada: el sistema retira el paquete de la zona actual, actualiza las coordenadas y exige una nueva clasificación manual por parte del almacenista.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST permitir al almacenista asignar una zona de almacenamiento física al paquete, vinculada a su UUID.
- **FR-002**: System MUST sugerir automáticamente la zona de almacenamiento correcta basándose en las coordenadas GPS del destino del paquete.
- **FR-003**: System MUST actualizar el estado del paquete a `En Clasificación` en el flujo exitoso, o a `Fuera de Tolerancia` cuando se detecten inconsistencias de peso o datos GPS inválidos.
- **FR-004**: System MUST emitir alertas de manejo especial para mercancías `Frágil` o `Peligroso` e impedir su asignación a zonas no aptas para su categoría.
- **FR-005**: System MUST bloquear la clasificación de paquetes sin coordenadas GPS completas y válidas, marcándolos como `Fuera de Tolerancia — Sin GPS`.
- **FR-006**: System MUST actualizar el contador de capacidad (peso acumulado y volumen acumulado) de la zona de almacenamiento al asignar o reasignar un paquete.
- **FR-007**: System MUST emitir alerta de zona saturada y sugerir zona de contingencia cuando la zona destino alcance su capacidad máxima configurada.

### Key Entities *(include if feature involves data)*

- **Paquete**: Entidad que se clasifica. Debe tener UUID, coordenadas GPS y datos físicos completos para poder ser procesada.
- **Zona de Almacenamiento**: Espacio físico en bodega con atributos UUID, nombre, categoría (Normal, Delicada, Alto Riesgo, Retención), capacidad_max_kg, capacidad_max_m3, peso_actual_kg, volumen_actual_m3 y estado (Disponible, Parcial, Saturado).
- **Almacenista**: Usuario responsable de la clasificación física cuyas acciones quedan auditadas en el historial de estados del paquete.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los paquetes que completan el proceso deben tener una zona de almacenamiento asignada; ninguno puede quedar en estado `En Clasificación` sin zona registrada.
- **SC-002**: El 0% de los paquetes `Frágil` o `Peligroso` pueden ser asignados a zonas no aptas para su categoría de mercancía.
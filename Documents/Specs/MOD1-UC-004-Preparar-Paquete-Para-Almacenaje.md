# Feature Specification: Preparar Paquete para Almacenaje (MOD1-UC-004)

**Created**: 2026-02-28

## User Scenarios & Testing

### User Story 4 — Asignación de zona de almacenamiento física (P1)

Como Almacenista, necesito asignar el paquete a su zona en bodega para iniciar su procesamiento interno e invocar automáticamente la clasificación por zona de destino.

**Why this priority**: Es la primera acción en bodega. La solicitud de ruta ya fue emitida durante la admisión; la responsabilidad aquí es la correcta organización física.

**Importante**: La selección del paquete se realiza por UUID. Si el paquete presenta discrepancias físicas (la información real del paquete no coincide con la registrada en el sistema), el Almacenista puede actualizar los datos directamente.

**Independent Test**: Seleccionar un paquete por UUID en `Recibido en Sede` con datos completos, confirmar la zona y verificar que el estado cambie a `En Clasificación`, los contadores de la zona se actualicen y se invoque [Clasificar Paquete por Zona de Destino (MOD1-UC-005)](./MOD1-UC-005-Clasificar-Paquete-Por-Zona-Destino.md).

**Acceptance Scenarios**:

1. **Asignación exitosa - (Paquete en buen estado)**
   - **Given** el paquete está en `Recibido en Sede` con datos completos, y se encuentra en buen estado físico.
   - **When** el almacenista selecciona el paquete por UUID y confirma la zona sugerida.
   - **Then** el sistema registra la zona, actualiza los contadores de capacidad (peso, volumen y cantidad de paquetes), cambia el estado a `En Clasificación` e invoca [Clasificar Paquete por Zona de Destino (MOD1-UC-005)](./MOD1-UC-005-Clasificar-Paquete-Por-Zona-Destino.md).

2. **Alerta de manejo especial**
   - **Given** el paquete es de tipo `Frágil` o `Peligroso`.
   - **When** el almacenista selecciona el paquete por UUID.
   - **Then** el sistema emite alerta de manejo especial y muestra solo las zonas aptas para esa categoría.

3. **Actualización directa de datos por discrepancia física**
   - **Given** el paquete presenta discrepancias físicas (peso o dimensiones) que no coinciden con los datos registrados en el sistema.
   - **When** el almacenista detecta la discrepancia y registra los nuevos datos correctos directamente en el sistema.
   - **Then** el sistema actualiza los datos del paquete, registra la corrección en el historial con identificador del almacenista, y el flujo continúa normalmente.

4. **Paquete dañado o extraviado (Novedad)**
   - **Given** el paquete se encuentra dañado o extraviado durante la preparación.
   - **When** el almacenista detecta la novedad.
   - **Then** el sistema interrumpe el flujo normal de almacenaje e invoca el caso de uso [Actualizar Estado de Paquete por Novedad (MOD1-UC-006)](./MOD1-UC-006-Actualizar-Estado-De-Paquete-Por-Novedad.md) para registrar la novedad con evidencia.

### Edge Cases

- **¿Qué ocurre si la zona de almacenamiento sugerida ya ha alcanzado su capacidad máxima?** El sistema sugiere automáticamente la zona de contingencia configurada y marca el paquete como desborde de zona, permitiendo continuar el flujo sin interrupciones.
- **¿Cómo maneja el sistema el intento de dos almacenistas de procesar el mismo paquete simultáneamente?** La primera operación recibida tiene prioridad (FIFO). El segundo almacenista recibe un mensaje indicando que el paquete ya fue procesado.
- **¿Qué sucede si el almacenista intenta asignar un paquete de tipo `Peligroso` a una zona de categoría Normal?** El sistema bloquea la asignación con un error explícito e impide continuar hasta que se seleccione una zona de categoría `Alto Riesgo`.
- **¿Qué pasa si el almacenista intenta procesar un paquete cuyo UUID no existe en el sistema?** El sistema informa que el UUID no fue encontrado y bloquea la operación hasta que se ingrese un identificador válido.

---

## Requirements

### Functional Requirements

- **FR-001**: Permitir al almacenista asignar una zona de almacenamiento al paquete por su UUID. La selección del paquete se realiza exclusivamente por UUID.
- **FR-002**: Sugerir automáticamente la zona más apropiada según el tipo de mercancía y las coordenadas de destino.
- **FR-003**: Actualizar el estado a `En Clasificación` en flujo exitoso (cuando el paquete está en buen estado).
- **FR-004**: Permitir al Almacenista actualizar datos del paquete directamente cuando se detecten discrepancias físicas (datos del paquete que no coinciden con el registro, peso, dimensiones), registrando la corrección en el historial.
- **FR-005**: Si el paquete está dañado o extraviado durante la preparación, interrumpir el flujo normal e invocar [Actualizar Estado de Paquete por Novedad (MOD1-UC-006)](./MOD1-UC-006-Actualizar-Estado-De-Paquete-Por-Novedad.md).
- **FR-006**: Bloquear la asignación de mercancía `Frágil` o `Peligrosa` a zonas no aptas.
- **FR-007**: Actualizar atómicamente los tres contadores de la zona (peso acumulado, volumen acumulado y cantidad de paquetes) al asignar o retirar un paquete.
- **FR-008**: Emitir alerta de zona saturada y sugerir zona de contingencia al alcanzar cualquiera de los límites configurados.
- **FR-009**: Invocar [Clasificar Paquete por Zona de Destino (MOD1-UC-005)](./MOD1-UC-005-Clasificar-Paquete-Por-Zona-Destino.md) automáticamente al confirmar el cambio a `En Clasificación` en el flujo exitoso.

### Key Entities

- **Zona de Almacenamiento**: nombre, categoría (Normal / Delicada / Alto Riesgo / Retención), capacidades máximas y valores actuales de peso, volumen y cantidad de paquetes, estado (Disponible / Parcial / Saturado).

---

## Success Criteria

- **SC-001**: El 100% de los paquetes en buen estado tienen zona asignada y contadores actualizados correctamente.
- **SC-002**: El 0% de los paquetes `Frágil` o `Peligroso` son asignados a zonas de categoría Normal.
- **SC-003**: El 100% de las discrepancias físicas detectadas se registran con actualización directa de datos.
- **SC-004**: El 100% de los paquetes dañados o extraviados interrumpen el flujo e invocan el caso de uso de gestión de novedades.
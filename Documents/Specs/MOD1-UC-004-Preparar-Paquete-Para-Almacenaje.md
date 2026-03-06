# Feature Specification: Preparar Paquete para Almacenaje (MOD1-UC-004)

**Created**: 2026-02-28

## User Scenarios & Testing

### User Story 1 — Asignación de zona de almacenamiento física (P1)

Como Almacenista, necesito asignar el paquete a su zona en bodega para iniciar su procesamiento interno e invocar automáticamente la clasificación por zona de destino.

**Why this priority**: Es la primera acción en bodega. La solicitud de ruta ya fue emitida durante la admisión; la responsabilidad aquí es la correcta organización física.

**Independent Test**: Escanear el UUID de un paquete en `Recibido en Sede` con datos completos, confirmar la zona y verificar que el estado cambie a `En Clasificación`, los contadores de la zona se actualicen y se invoque [MOD1-UC-006](./MOD1-UC-006-Clasificar-Paquete-Por-Zona-Destino.md).

**Acceptance Scenarios**:

1. **Asignación exitosa**
   - **Given** el paquete está en `Recibido en Sede` con GPS resuelto y datos físicos completos.
   - **When** el almacenista escanea el UUID y confirma la zona sugerida.
   - **Then** el sistema registra la zona, actualiza los contadores de capacidad (peso, volumen y cantidad de paquetes), cambia el estado a `En Clasificación` e invoca [MOD1-UC-006](./MOD1-UC-006-Clasificar-Paquete-Por-Zona-Destino.md).

2. **Alerta de manejo especial**
   - **Given** el paquete es de tipo `Frágil` o `Peligroso`.
   - **When** el almacenista escanea el UUID.
   - **Then** el sistema emite alerta de manejo especial y muestra solo las zonas aptas para esa categoría.

### Edge Cases

- **Zona saturada**: el sistema sugiere la zona de contingencia configurada; el paquete queda marcado como desborde de zona.
- **Diferencia de peso > 10%**: el paquete pasa a `Fuera de Tolerancia — Peso` y queda bloqueado hasta que el Supervisor de Bodega apruebe el nuevo valor.
- **GPS pendiente**: el sistema bloquea la asignación y marca `Fuera de Tolerancia — Sin GPS`.
- **Dos almacenistas simultáneos**: la segunda operación recibe un mensaje indicando que el paquete ya fue procesado (FIFO).
- **Peligroso en zona normal**: el sistema bloquea la asignación con un error explícito.
- **Cambio de dirección posterior**: se retira el paquete de la zona (contadores revertidos), se actualiza la dirección y, si aplica, se re-emite `solicitar_ruta` bajo autorización del Supervisor.

---

## Requirements

### Functional Requirements

- **FR-001**: Permitir al almacenista asignar una zona de almacenamiento al paquete por su UUID.
- **FR-002**: Sugerir automáticamente la zona más apropiada según el tipo de mercancía y las coordenadas de destino.
- **FR-003**: Actualizar el estado a `En Clasificación` en flujo exitoso, o a `Fuera de Tolerancia` ante inconsistencias de peso o GPS.
- **FR-004**: Bloquear la asignación de mercancía `Frágil` o `Peligrosa` a zonas no aptas.
- **FR-005**: Bloquear paquetes con GPS pendiente y marcarlos como `Fuera de Tolerancia — Sin GPS`.
- **FR-006**: Actualizar atómicamente los tres contadores de la zona (peso acumulado, volumen acumulado y cantidad de paquetes) al asignar o retirar un paquete.
- **FR-007**: Emitir alerta de zona saturada y sugerir zona de contingencia al alcanzar cualquiera de los límites configurados.
- **FR-008**: Invocar [MOD1-UC-006](./MOD1-UC-006-Clasificar-Paquete-Por-Zona-Destino.md) automáticamente al confirmar el cambio a `En Clasificación`.

### Key Entities

- **Zona de Almacenamiento**: nombre, categoría (Normal / Delicada / Alto Riesgo / Retención), capacidades máximas y valores actuales de peso, volumen y cantidad de paquetes, estado (Disponible / Parcial / Saturado).

---

## Success Criteria

- **SC-001**: El 100% de los paquetes procesados tienen zona asignada y contadores actualizados correctamente.
- **SC-002**: El 0% de los paquetes `Frágil` o `Peligroso` son asignados a zonas de categoría Normal.
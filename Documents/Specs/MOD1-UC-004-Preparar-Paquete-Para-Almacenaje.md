# Feature Specification: Preparar Paquete para Almacenaje (MOD1-UC-004)

**Created**: 2026-02-28

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Asignación de zona de almacenamiento física en bodega (Priority: P1)

Como Almacenista, necesito ubicar físicamente el paquete en su zona de almacenamiento correspondiente dentro de la bodega, para que el sistema registre la ubicación, invoque la clasificación por zona de destino lógico e inicie el seguimiento interno del paquete dentro de las operaciones de bodega.

**Why this priority**: Es la primera acción del almacenista sobre el paquete. En este punto, la solicitud de ruta ya fue emitida al `Módulo de Gestión de Rutas` durante la admisión (MOD1-UC-001 + MOD1-UC-002 + MOD1-UC-003). La responsabilidad de este caso de uso es la correcta organización física en bodega y la clasificación lógica del paquete. Invoca [Clasificar Paquete por Zona de Destino (MOD1-UC-006)](./MOD1-UC-006-Clasificar-Paquete-Por-Zona-Destino.md).

**Independent Test**: Puede probarse escaneando el UUID de un paquete en estado `Recibido en Sede` con datos físicos y GPS completos, confirmando la zona de almacenamiento sugerida y verificando que: (1) el estado cambia a `En Clasificación`, (2) se actualizan los contadores de ocupación de la zona (`peso_actual_kg`, `volumen_actual_m3`, `contador_paquetes`) y (3) se invoca MOD1-UC-006 para asignar la zona de destino lógico.

**Acceptance Scenarios**:

1. **Scenario**: Asignación exitosa a zona de almacenamiento
   - **Given** el paquete tiene estado `Recibido en Sede`, `gps_estado = Resuelto`, datos físicos completos (`peso_kg`, `volumen_m3`, `tipo_mercancia`) y el almacenista está autenticado.
   - **When** el almacenista escanea el UUID y el sistema sugiere la zona de almacenamiento basada en las coordenadas GPS de destino y el `tipo_mercancia`; el almacenista confirma la ubicación.
   - **Then** el sistema registra la zona de almacenamiento asignada al paquete, actualiza los contadores de capacidad de la zona (`peso_actual_kg += peso_kg`, `volumen_actual_m3 += volumen_m3`, `contador_paquetes += 1`), actualiza el estado del paquete a `En Clasificación` e invoca automáticamente [Clasificar Paquete por Zona de Destino (MOD1-UC-006)](./MOD1-UC-006-Clasificar-Paquete-Por-Zona-Destino.md).

2. **Scenario**: Alerta de manejo especial para mercancía frágil o peligrosa
   - **Given** el paquete tiene `tipo_mercancia = Frágil` o `Peligroso`.
   - **When** el almacenista escanea el UUID.
   - **Then** el sistema emite una alerta de manejo especial y sugiere únicamente zonas de almacenamiento de categoría apta (`Delicada` para `Frágil`, `Alto Riesgo` para `Peligroso`). La asignación a zonas `Normal` queda bloqueada.

---

### Edge Cases

- What happens when la zona de almacenamiento está saturada (`peso_actual_kg` o `volumen_actual_m3` o `contador_paquetes` han alcanzado su máximo)? El sistema detecta la saturación, emite la alerta `Zona de Almacenamiento Saturada` y sugiere automáticamente la zona de contingencia más cercana disponible. El paquete asignado a la zona alternativa queda marcado con nota `Desborde de zona`. El flujo continúa normalmente con la zona alternativa.
- What happens when el peso físico observado en bodega difiere del `peso_kg` registrado en admisión en más de un 10%? El sistema marca el paquete como `Fuera de Tolerancia — Peso`, bloquea el cambio a `En Clasificación` y notifica al Supervisor de Bodega para que corrija y apruebe el nuevo valor. Una vez aprobado, si la diferencia afecta el payload ya enviado al `Módulo de Gestión de Rutas`, se re-emite [Solicitar Ruta de Paquete (MOD1-UC-003)](./MOD1-UC-003-Solicitar-Ruta-De-Paquete.md) con los datos corregidos.
- What happens when el paquete tiene `gps_estado = Pendiente GPS`? El sistema impide la asignación a zona de almacenamiento, marca el paquete `Fuera de Tolerancia — Sin GPS` y notifica al Empleado de Envío para resolver las coordenadas. Las coordenadas son un campo obligatorio para la clasificación por zona de destino lógico.
- What happens when el `Módulo de Gestión de Rutas` no ha respondido aún la solicitud de ruta emitida durante la admisión? El paquete puede ser asignado a zona de almacenamiento normalmente. La asignación de zona de almacenamiento y la respuesta del M2 son procesos independientes. El paquete avanza a `En Clasificación` independientemente del estado de la Solicitud de Ruta.
- How does system handle si dos almacenistas intentan procesar el mismo paquete simultáneamente? El sistema aplica control de concurrencia optimista (versioning de entidad). La primera operación confirma exitosamente. La segunda operación detecta el conflicto de versión y notifica al segundo almacenista que el paquete ya fue procesado.
- What happens when un paquete `Peligroso` intenta asignarse a una zona `Normal`? El sistema bloquea la asignación con error explícito. No es posible confirmar hasta seleccionar una zona de categoría `Alto Riesgo`.
- How does system handle el cambio de dirección de destino solicitado después de que el paquete fue asignado a zona de almacenamiento? El cambio es una excepción supervisada: el sistema retira el paquete de la zona actual (actualizando contadores de capacidad en sentido inverso), actualiza coordenadas GPS y dirección, exige nueva clasificación manual y, si aplica, re-emite [Solicitar Ruta de Paquete (MOD1-UC-003)](./MOD1-UC-003-Solicitar-Ruta-De-Paquete.md) con los datos corregidos bajo autorización del Supervisor de Bodega.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST permitir al almacenista asignar una zona de almacenamiento física al paquete, vinculada a su UUID.
- **FR-002**: System MUST sugerir automáticamente la zona de almacenamiento más apropiada basándose en las coordenadas GPS de destino y el `tipo_mercancia` del paquete.
- **FR-003**: System MUST actualizar el estado del paquete a `En Clasificación` en el flujo exitoso, o a `Fuera de Tolerancia` (subtipo `Peso` o `Sin GPS`) cuando se detecten inconsistencias bloqueantes.
- **FR-004**: System MUST emitir alertas de manejo especial para `Frágil` o `Peligroso` e impedir la asignación a zonas de categoría `Normal`.
- **FR-005**: System MUST bloquear la clasificación de paquetes con `gps_estado = Pendiente GPS`, marcándolos como `Fuera de Tolerancia — Sin GPS`.
- **FR-006**: System MUST actualizar de forma atómica los tres contadores de capacidad de la zona de almacenamiento al confirmar la asignación: `peso_actual_kg += peso_kg`, `volumen_actual_m3 += volumen_m3`, `contador_paquetes += 1`. Al retirar un paquete de una zona (reasignación o corrección), los contadores deben decrementarse de forma igualmente atómica.
- **FR-007**: System MUST emitir la alerta `Zona de Almacenamiento Saturada` y sugerir una zona de contingencia disponible cuando cualquiera de los tres contadores (`peso_actual_kg`, `volumen_actual_m3`, `contador_paquetes`) alcance su valor máximo configurado.
- **FR-008**: System MUST invocar automáticamente [Clasificar Paquete por Zona de Destino (MOD1-UC-006)](./MOD1-UC-006-Clasificar-Paquete-Por-Zona-Destino.md) tras confirmar exitosamente el cambio de estado a `En Clasificación`.

### Key Entities *(include if feature involves data)*

- **Paquete**: En este punto debe tener: `uuid`, `gps_estado = Resuelto`, `latitud`, `longitud`, `peso_kg`, `volumen_m3`, `tipo_mercancia`, `metodo_pago` y `valor_declarado`.
- **Zona de Almacenamiento**: Espacio físico en bodega. Atributos: `id` (PK), `nombre`, `categoria` (`Normal` | `Delicada` | `Alto Riesgo` | `Retención`), `capacidad_max_kg`, `capacidad_max_m3`, `capacidad_max_paquetes`, `peso_actual_kg`, `volumen_actual_m3`, `contador_paquetes`, `estado` (`Disponible` | `Parcial` | `Saturado`).
- **Almacenista**: Usuario con rol `Almacenista` o superior. Acciones auditadas en el Historial de Estados.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los paquetes que completan este proceso deben tener zona de almacenamiento asignada y los contadores de la zona actualizados de forma consistente.
- **SC-002**: El 0% de los paquetes `Frágil` o `Peligroso` pueden ser asignados a zonas de categoría `Normal`.
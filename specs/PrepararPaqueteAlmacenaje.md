# Feature Specification: Preparar Paquete para Almacenaje (MOD1-UC-003)

**Created**: 2026-02-28

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Organización física en bodega y solicitud de ruta al Módulo 2 (Priority: P1)

Como Almacenista, necesito ubicar físicamente el paquete en su zona de almacenamiento correspondiente para que, al quedar clasificado, el sistema emita automáticamente la solicitud de ruta al Módulo 2 con todos los datos ya completos del paquete.

**Why this priority**: Es el punto exacto del ciclo de vida donde el paquete pasa a `En Clasificación`, estado en el que ya cuenta con todos los datos requeridos por el Contrato de Integración (peso, volumen, tipo de mercancía, coordenadas GPS, método de pago y valor declarado). Esto lo convierte en el momento correcto para disparar el evento `solicitar_ruta` al Módulo 2, de forma que la planificación de rutas comience en paralelo mientras el paquete es ubicado físicamente en bodega. Además, invoca `Clasificar Paquete por Zona de Destino`.

**Independent Test**: Puede probarse escaneando el UUID de un paquete en estado `Recibido en Sede` con datos físicos y GPS completos, confirmando la zona sugerida y verificando que: (1) el estado cambia a `En Clasificación`, (2) se actualiza el contador de ocupación de la zona y (3) el evento `solicitar_ruta` es emitido al Módulo 2 con el payload completo según el Contrato de Integración.

**Acceptance Scenarios**:

1. **Scenario**: Asignación exitosa a zona y emisión de solicitud de ruta
   - **Given** el paquete existe con estado `Recibido en Sede`, tiene coordenadas GPS válidas y datos físicos registrados (peso, volumen, tipo de mercancía), y el almacenista está autenticado.
   - **When** el almacenista escanea el UUID, el sistema sugiere automáticamente la zona de almacenamiento y el almacenista confirma la ubicación.
   - **Then** el sistema invoca `Clasificar Paquete por Zona de Destino`, registra la zona asignada, actualiza el estado del paquete a `En Clasificación`, actualiza el contador de ocupación de la zona e invoca automáticamente `Solicitar Ruta de Paquete`, que construye y envía el evento `solicitar_ruta` al Módulo 2 con el payload completo.

2. **Scenario**: Alerta de manejo especial para mercancía frágil o peligrosa
   - **Given** el paquete tiene el tipo de mercancía registrado como `Frágil` o `Peligroso`.
   - **When** el almacenista escanea el UUID para iniciar la preparación de almacenaje.
   - **Then** el sistema emite una alerta de manejo especial, sugiere únicamente zonas aptas para esa categoría (`Zona de Manipulación Delicada` o `Zona de Resguardo de Alto Riesgo`) e impide la asignación a zonas no calificadas.

---

### Edge Cases

- What happens when la zona de destino está saturada (capacidad máxima alcanzada)? El sistema detecta la condición, actualiza el estado de la zona a `Saturado`, emite una alerta de `Zona Saturada` y sugiere automáticamente una zona de contingencia disponible. El paquete asignado a la zona alternativa queda marcado con nota de `Desborde de zona`. La solicitud de ruta al Módulo 2 se emite igualmente una vez confirmada la zona alternativa.
- What happens when el peso físico del paquete en bodega difiere significativamente del registrado en admisión? Si la diferencia supera el umbral configurado (10%), el sistema marca el paquete como `Fuera de Tolerancia`, bloqueando el cambio de estado a `En Clasificación` y la emisión del evento `solicitar_ruta` hasta que un supervisor resuelva la inconsistencia.
- What happens when el paquete no tiene coordenadas GPS válidas al momento de escanear? El sistema impide la clasificación y la solicitud de ruta, marca el paquete como `Fuera de Tolerancia — Sin GPS` y notifica al Empleado de Envío para que resuelva el dato geográfico. Las coordenadas son obligatorias en el payload del Contrato de Integración.
- What happens when el Módulo 2 rechaza el evento `solicitar_ruta` por campo inválido? El sistema revierte el estado del paquete a `Recibido en Sede`, notifica al almacenista con el campo específico que causó el rechazo y habilita la corrección para reintentar el flujo.
- What happens when el Módulo 2 no responde? El evento se encola para reintento automático sin bloquear el paquete. El estado `En Clasificación` se mantiene y el almacenista puede continuar con la ubicación física mientras se resuelve la comunicación.
- How does system handle si dos almacenistas intentan procesar el mismo paquete simultáneamente? El sistema aplica bloqueo optimista: la segunda operación detecta el conflicto y notifica al segundo almacenista que el paquete ya fue procesado, mostrando el estado actual.
- What happens when un paquete marcado como `Peligroso` intenta ser asignado a una zona normal? El sistema bloquea la asignación, muestra un error explícito e impide confirmar hasta que se seleccione una zona apta para mercancía peligrosa.
- How does system handle si el cliente cambia la dirección de destino después de que el paquete ya fue clasificado? El cambio se gestiona como excepción supervisada: el sistema retira el paquete de la zona actual, actualiza las coordenadas, exige una nueva clasificación manual y re-emite el evento `solicitar_ruta` con los datos corregidos.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST permitir al almacenista asignar una zona de almacenamiento física al paquete, vinculada a su UUID.
- **FR-002**: System MUST sugerir automáticamente la zona de almacenamiento correcta basándose en las coordenadas GPS del destino del paquete.
- **FR-003**: System MUST actualizar el estado del paquete a `En Clasificación` en el flujo exitoso, o a `Fuera de Tolerancia` cuando se detecten inconsistencias de peso o datos GPS inválidos.
- **FR-004**: System MUST emitir alertas de manejo especial para mercancías `Frágil` o `Peligroso` e impedir su asignación a zonas no aptas para su categoría.
- **FR-005**: System MUST bloquear la clasificación y la solicitud de ruta de paquetes sin coordenadas GPS completas y válidas, marcándolos como `Fuera de Tolerancia — Sin GPS`.
- **FR-006**: System MUST actualizar el contador de capacidad (peso acumulado y volumen acumulado) de la zona de almacenamiento al asignar o reasignar un paquete.
- **FR-007**: System MUST emitir alerta de zona saturada y sugerir zona de contingencia cuando la zona destino alcance su capacidad máxima configurada.
- **FR-008**: System MUST invocar automáticamente el caso de uso `Solicitar Ruta de Paquete` al confirmar exitosamente el cambio de estado a `En Clasificación`, el cual emitirá el evento `solicitar_ruta` al Módulo 2 con el payload completo del Contrato de Integración.
- **FR-009**: System MUST revertir el estado a `Recibido en Sede` si el Módulo 2 rechaza el evento `solicitar_ruta`, notificando al almacenista con el detalle del campo inválido.

### Key Entities *(include if feature involves data)*

- **Paquete**: Entidad que se clasifica. Debe tener UUID, coordenadas GPS y datos físicos completos (peso_kg, volumen_m3, tipo_mercancia, metodo_pago, valor_declarado) para poder ser procesada y para que el payload de `solicitar_ruta` sea válido.
- **Zona de Almacenamiento**: Espacio físico en bodega con atributos UUID, nombre, categoría (Normal, Delicada, Alto Riesgo, Retención), capacidad_max_kg, capacidad_max_m3, peso_actual_kg, volumen_actual_m3 y estado (Disponible, Parcial, Saturado).
- **Almacenista**: Usuario responsable de la clasificación física cuyas acciones quedan auditadas en el historial de estados del paquete.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los paquetes que completan el proceso deben tener una zona de almacenamiento asignada y haber disparado el evento `solicitar_ruta` al Módulo 2.
- **SC-002**: El 0% de los paquetes `Frágil` o `Peligroso` pueden ser asignados a zonas no aptas para su categoría de mercancía.
# Feature Specification: Preparar Paquete para Almacenaje (MOD1-UC-003)

**Created**: 2026-02-26

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Organización física en bodega (Priority: P1-Crítica)

Como Almacenista, necesito organizar físicamente los paquetes en las zonas de almacenamiento sugeridas según su destino y tipo de mercancía.

**Why this priority**: Garantiza que el Módulo 2 pueda realizar la consolidación de carga de forma eficiente eliminando los puntos ciegos físicos en bodega.

**Independent Test**: Puede probarse escaneando un UUID, verificando que el sistema sugiera la zona correcta según las coordenadas GPS y el tipo de mercancía, y que al confirmar se actualice el contador de capacidad.

**Acceptance Scenarios**:

1. **Scenario**: Asignación exitosa a zona sugerida
   - **Given**: El paquete existe con estado 'Recibido en Sede', tiene coordenadas GPS y datos físicos registrados, y el almacenista está autenticado.
   - **When**: El almacenista escanea el UUID y el sistema sugiere automáticamente la zona de almacenamiento.
   - **Then**: El almacenista confirma, el sistema registra la zona, actualiza el estado a 'En Clasificación' y actualiza el contador de ocupación.

### Edge Cases

- What happens when: la zona de destino está saturada? El sistema detecta la capacidad máxima, emite una alerta de 'Zona Saturada', sugiere automáticamente una zona de contingencia y añade una nota de desborde.
- What happens when: el peso físico detectado en bodega difiere del registrado en admisión? Si la diferencia supera el umbral configurado (10%), el sistema marca el paquete como 'Fuera de Tolerancia', bloqueando el despacho hasta resolución.
- What happens when: las coordenadas GPS son faltantes o inválidas? El sistema impide la clasificación y marca el paquete como 'Fuera de Tolerancia Sin GPS'.
- How does system handle: si un paquete cambia de zona mientras otro almacenista lo procesa simultáneamente? El sistema usa bloqueo optimista y notifica al segundo almacenista que el paquete ya fue procesado.
- What happens when: un paquete marcado como 'Peligroso' intenta ser asignado a una zona normal? El sistema bloquea la asignación mostrando un error y exige seleccionar una zona apta.
- How does system handle: si el cliente cambia la dirección de destino después de la clasificación? El sistema requiere que el cambio se gestione como una excepción supervisada, retirando el paquete de la zona actual y exigiendo reclasificación manual.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST: permitir al almacenista asignar una zona de almacenamiento física al paquete, vinculada a su UUID.
- **FR-002**: System MUST: sugerir automáticamente la zona de almacenamiento basándose en las coordenadas GPS del destino.
- **FR-003**: System MUST: actualizar el estado del paquete a 'En Clasificación' o 'Fuera de Tolerancia' según las condiciones.
- **FR-004**: System MUST: emitir alertas de manejo especial para mercancías frágiles o peligrosas e impedir su asignación a zonas no aptas.
- **FR-005**: System MUST: bloquear la clasificación de paquetes sin coordenadas GPS válidas marcándolos como 'Fuera de Tolerancia'.
- **FR-006**: System MUST: actualizar el contador de capacidad (peso y volumen) de la zona al asignar o reasignar un paquete.
- **FR-007**: System MUST: emitir alerta de zona saturada y sugerir zona de contingencia cuando alcance su capacidad máxima.
- **NFR-001**: System MUST: garantizar que el tiempo de respuesta tras escanear el UUID hasta mostrar la zona no supere los 3 segundos.
- **NFR-002**: System MUST: proporcionar una interfaz operable exclusivamente con lector de código de barras, sin necesidad de teclado.

### Key Entities *(include if feature involves data)*

- **Paquete**: Entidad que se clasifica, requiriendo UUID, coordenadas GPS y datos físicos completos.
- **Zona de Almacenamiento**: Espacio físico en bodega con atributos de categoría, capacidad máxima y estado de ocupación.
- **Almacenista**: Usuario responsable de la clasificación física cuyas acciones quedan auditadas.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los paquetes que completan el proceso deben tener una zona de almacenamiento asignada; ninguno puede quedar 'En Clasificación' sin zona.
- **SC-002**: El tiempo de respuesta del sistema al escanear el UUID no debe superar los 3 segundos en el percentil 95.
- **SC-003**: El 0% de los paquetes frágiles o peligrosos pueden ser asignados a zonas no aptas para su categoría.
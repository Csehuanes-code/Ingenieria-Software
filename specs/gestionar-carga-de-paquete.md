# Feature Specification: Gestionar Carga de Paquete (MOD1-UC-006)

**Created**: 2026-02-26

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Confirmación de salida en manifiesto (Priority: P2-Alta)

Como Despachador de Carga, necesito escanear y verificar los paquetes listados en el manifiesto para confirmar formalmente su salida física en ruta.

**Why this priority**: Es el último punto de control del Módulo 1 antes de que el Módulo 2 asuma la custodia completa de los paquetes en campo.

**Independent Test**: Puede probarse simulando la recepción de un manifiesto, escaneando los paquetes, y verificando que su estado cambia a 'En Tránsito' mientras el sistema notifica al Módulo 2 de la salida confirmada.

**Acceptance Scenarios**:

1. **Scenario**: Confirmación de carga estándar
   - **Given**: El paquete está en estado 'Listo para Despacho', ha sido asignado a una ruta por el Módulo 2 y el despachador está autenticado.
   - **When**: El despachador accede al manifiesto, escanea cada UUID para verificar presencia física y confirma la carga.
   - **Then**: El sistema actualiza el estado a 'En Tránsito', registra la fecha de salida, asocia el ID del despachador y notifica al Módulo 2.

### Edge Cases

- What happens when: un paquete del manifiesto no se encuentra físicamente presente en bodega? El despachador lo marca como 'No Encontrado', el paquete permanece 'Listo para Despacho' generando alerta, y se puede proceder con despacho parcial si se autoriza.
- What happens when: la sesión del despachador expira durante la operación de escaneo masivo? El sistema guarda el progreso de los paquetes ya verificados, permitiendo continuar al reiniciar la sesión sin necesidad de re-escaneo.
- How does system handle: si dos despachadores intentan gestionar el mismo paquete simultáneamente? El sistema bloquea la segunda operación e informa qué despachador ya está gestionando el paquete.
- What happens when: un paquete aparece con estado 'Fuera de Tolerancia' dentro del manifiesto? El sistema impide su escaneo, muestra el motivo del bloqueo y requiere escalamiento al almacenista antes del despacho.
- What happens when: un paquete marcado como 'Dañado' o 'Extraviado' es detectado en el manifiesto? El sistema alerta al despachador, impide la carga, lo retira automáticamente del manifiesto activo y escala el caso al Controlador de Novedades.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST: permitir la gestión de carga únicamente para paquetes con estado 'Listo para Despacho'.
- **FR-002**: System MUST: actualizar el estado a 'En Tránsito' únicamente tras la confirmación explícita del despachador de carga.
- **FR-003**: System MUST: registrar automáticamente la fecha/hora en formato timestamp UTC de la gestión de carga.
- **FR-004**: System MUST: asociar permanentemente el ID del Despachador responsable a cada operación de carga.
- **FR-005**: System MUST: impedir operaciones de gestión de carga duplicadas sobre el mismo UUID mediante control de concurrencia.
- **FR-006**: System MUST: soportar la gestión de carga parcial, permitiendo que algunos paquetes salgan mientras otros quedan pendientes.
- **NFR-001**: System MUST: promediar un tiempo de gestión de carga (escaneo y confirmación) no superior a 3 minutos por paquete.
- **NFR-002**: System MUST: garantizar que un UUID no pueda transicionar al estado 'En Tránsito' más de una vez sin haber regresado previamente por un flujo de novedad.

### Key Entities *(include if feature involves data)*

- **Manifiesto de Ruta**: Listado de paquetes asignados a un vehículo, generado por el Módulo 2 y consumido por el Despachador.
- **Registro de Carga**: Entidad que almacena el UUID del paquete, timestamp de carga, ID del despachador y el ID de la ruta.
- **Despachador de Carga**: Actor que ejecuta operativamente y confirma la salida física.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los paquetes gestionados en la carga deben transicionar correctamente al estado 'En Tránsito'.
- **SC-002**: El sistema debe bloquear satisfactoriamente el 100% de los intentos de gestión sobre paquetes con estados logísticos no permitidos.
- **SC-003**: El tiempo promedio de gestión de carga no debe superar los 3 minutos por paquete en el percentil 95.
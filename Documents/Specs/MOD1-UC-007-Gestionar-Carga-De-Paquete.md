# Feature Specification: Gestionar Carga de Paquete (MOD1-UC-007)

**Created**: 2026-02-28

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Carga física de paquetes al vehículo y confirmación de despacho (Priority: P2)

Como Coordinador de Despacho, necesito escanear y verificar cada paquete en estado `Clasificado` con ruta asignada por el `Módulo de Gestión de Rutas`, cargarlo físicamente al vehículo correspondiente, actualizar su estado a través de `En Carga` hasta `Listo para Despacho`, y finalmente confirmar al `Módulo de Gestión de Rutas` que el vehículo está completamente cargado y listo para iniciar su ruta.

**Why this priority**: Es el punto formal de salida física de la sede. El coordinador es el último actor del `Módulo de Gestión de Paquetes` que tiene custodia sobre el paquete antes de que el `Módulo de Gestión de Rutas` tome el control en campo. La confirmación del vehículo cargado es el evento que autoriza a M2 a emitir el estado `En Tránsito` y activar el seguimiento en ruta.

**Independent Test**: Puede probarse seleccionando un paquete en estado `Clasificado` con `id_ruta` e `id_transportador` disponibles en la entidad Solicitud de Ruta, escaneando su UUID para iniciar la carga (`En Carga`), confirmando su ubicación en el vehículo (`Listo para Despacho`), y verificando que — una vez todos los paquetes de la ruta estén en `Listo para Despacho` — el sistema emita la notificación de `vehiculo_listo` al `Módulo de Gestión de Rutas`.

**Acceptance Scenarios**:

1. **Scenario**: Flujo completo de carga — Clasificado → En Carga → Listo para Despacho
   - **Given** el paquete tiene estado `Clasificado`, tiene `id_ruta` e `id_transportador` registrados en la entidad Solicitud de Ruta, y el Coordinador de Despacho está autenticado.
   - **When** el coordinador escanea el UUID del paquete al recogerlo de la zona de almacenamiento.
   - **Then** el sistema actualiza el estado del paquete a `En Carga`, registra `timestamp_inicio_carga` UTC e `id_coordinador` en el Registro de Carga.
   - **Luego**, cuando el coordinador ubica el paquete en el vehículo y confirma que vehículo, zona de destino y ruta son los correctos:
   - **Then** el sistema actualiza el estado a `Listo para Despacho` y registra `timestamp_cargado` UTC.

2. **Scenario**: Confirmación de despacho del vehículo completo
   - **Given** todos los paquetes asociados a una `id_ruta` específica han alcanzado el estado `Listo para Despacho`.
   - **When** el Coordinador de Despacho verifica la lista de paquetes de la ruta y confirma que el vehículo está completamente cargado.
   - **Then** el sistema emite la notificación `vehiculo_listo` al `Módulo de Gestión de Rutas` con el `id_ruta`, la lista de UUIDs cargados y el `timestamp_despacho` UTC. M2 procede a emitir el estado `En Tránsito` para cada paquete.

3. **Scenario**: Bloqueo por estado incorrecto
   - **Given** el paquete tiene un estado diferente a `Clasificado` (ej. `En Clasificación`, `Fuera de Tolerancia`, `Dañado`).
   - **When** el coordinador intenta escanear el UUID para iniciar la carga.
   - **Then** el sistema bloquea la operación y muestra un mensaje indicando el estado actual del paquete y que el estado requerido para iniciar la carga es `Clasificado`.

4. **Scenario**: Bloqueo por ruta no confirmada
   - **Given** el paquete tiene estado `Clasificado` pero la entidad Solicitud de Ruta no tiene `id_ruta` ni `id_transportador` (M2 aún no respondió la solicitud emitida durante la admisión).
   - **When** el coordinador intenta escanear el UUID.
   - **Then** el sistema bloquea la operación, muestra el aviso `Ruta pendiente de asignación por el Módulo de Gestión de Rutas` e indica al coordinador que debe esperar la confirmación de M2 antes de proceder con la carga.

---

### Edge Cases

- What happens when el coordinador detecta discrepancias entre el paquete escaneado y la ruta asignada (ej. zona de destino del paquete no coincide con la zona del vehículo)? El sistema bloquea la confirmación de carga al vehículo y genera una alerta de `Discrepancia de Ruta`. El coordinador debe escalar al Supervisor de Bodega para resolver antes de continuar.
- What happens when dos coordinadores intentan cargar el mismo paquete simultáneamente? El sistema aplica control de concurrencia con prioridad FIFO: la primera operación recibida procesa el escaneo exitosamente. La segunda recibe el mensaje: `"El paquete ya está siendo procesado por [nombre del primer coordinador] desde [timestamp]."` No se permite la segunda operación.
- What happens when la sesión del coordinador expira durante el proceso de carga (estado `En Carga`)? El paquete permanece en estado `En Carga` con el `id_coordinador` registrado. Al reautenticarse, el mismo coordinador puede continuar. Si transcurren más de 60 minutos en estado `En Carga` sin actividad, el sistema genera una alerta al Supervisor de Bodega.
- What happens cuando un paquete en estado `En Carga` presenta un daño físico descubierto durante la manipulación? El coordinador no puede continuar con la carga. Debe reportar la novedad a través de [Gestionar Novedad de Paquete (MOD1-UC-008)](./MOD1-UC-008-Gestionar-Novedad-De-Paquete.md). El sistema revierte el estado a `Clasificado` y descuenta el paquete de los contadores del vehículo.
- What happens when el `Módulo de Gestión de Rutas` no responde a la notificación `vehiculo_listo`? El sistema persiste el Registro de Carga del vehículo como completado localmente y encola la notificación para reintento automático. Genera una alerta visible al coordinador indicando que la confirmación con M2 está pendiente.
- What happens when no todos los paquetes de una ruta llegan a estado `Listo para Despacho` antes del horario límite de despacho? El coordinador puede emitir la notificación `vehiculo_listo` de forma parcial, indicando los UUIDs efectivamente cargados. Los paquetes no cargados permanecen en su estado actual y son reasignados por M2 a la próxima ruta disponible.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST permitir gestionar la carga únicamente de paquetes en estado `Clasificado` que tengan `id_ruta` e `id_transportador` registrados en la entidad Solicitud de Ruta.
- **FR-002**: System MUST actualizar el estado del paquete a `En Carga` al momento en que el coordinador escanea el UUID para recogerlo, registrando `timestamp_inicio_carga` (UTC) e `id_coordinador` en el Registro de Carga.
- **FR-003**: System MUST actualizar el estado del paquete a `Listo para Despacho` cuando el coordinador confirme que el paquete está físicamente ubicado en el vehículo correcto y haya verificado que vehículo, zona de destino y ruta coinciden con los datos del paquete.
- **FR-004**: System MUST registrar `timestamp_cargado` (UTC) e `id_coordinador` al momento en que el paquete alcanza el estado `Listo para Despacho`.
- **FR-005**: System MUST recuperar el `id_transportador` desde la entidad Solicitud de Ruta asociada al UUID del paquete y persistirlo en el Registro de Carga. Este dato es requerido por [Informar Estado de Paquete (MOD1-UC-009)](./MOD1-UC-009-Informar-Estado-De-Paquete.md).
- **FR-006**: System MUST impedir la gestión de carga duplicada sobre un mismo UUID usando control de concurrencia con prioridad FIFO.
- **FR-007**: System MUST habilitar al Coordinador de Despacho para verificar la lista de paquetes asociados a una `id_ruta`, mostrando el estado de carga de cada uno (`Clasificado`, `En Carga`, `Listo para Despacho`).
- **FR-008**: System MUST permitir al Coordinador de Despacho confirmar el despacho del vehículo cuando todos o un subconjunto de paquetes de la ruta hayan alcanzado `Listo para Despacho`. Al confirmar, el sistema emite la notificación `vehiculo_listo` al `Módulo de Gestión de Rutas` con `id_ruta`, lista de UUIDs cargados y `timestamp_despacho` UTC.
- **FR-009**: System MUST encolar la notificación `vehiculo_listo` para reintento automático si M2 no responde, sin revertir ningún estado de paquete.

### Key Entities *(include if feature involves data)*

- **Paquete**: Transiciona `Clasificado` → `En Carga` → `Listo para Despacho` en este caso de uso.
- **Solicitud de Ruta**: Fuente del `id_ruta` e `id_transportador`, creados en MOD1-UC-003.
- **Registro de Carga**: `id` (PK), `uuid_paquete` (FK), `id_ruta` (FK), `id_coordinador` (FK), `id_transportador`, `timestamp_inicio_carga` (UTC), `timestamp_cargado` (UTC), `estado_notificacion_m2` (`Pendiente` | `Confirmado` | `Error`).
- **Registro de Despacho de Vehículo**: `id` (PK), `id_ruta` (FK), `id_coordinador` (FK), `uuids_cargados` (lista), `timestamp_despacho` (UTC), `estado_notificacion_m2` (`Pendiente` | `Confirmado` | `Error`).
- **Coordinador de Despacho**: Usuario con rol `Coordinador de Despacho`.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los paquetes gestionados deben generar un Registro de Carga con `uuid_paquete`, `id_transportador`, `timestamp_inicio_carga`, `timestamp_cargado` e `id_coordinador` completos.
- **SC-002**: El sistema debe bloquear el 100% de los intentos de carga sobre paquetes en estados distintos a `Clasificado` o sin `id_ruta` disponible.
- **SC-003**: El tiempo promedio del proceso de carga por paquete (escaneo + confirmación en vehículo) no debe superar los 3 minutos en el percentil 95.
# Feature Specification: Gestionar Carga de Paquete (MOD1-UC-007)

**Created**: 2026-02-28

## User Scenarios & Testing

### User Story 1 — Carga física y despacho del vehículo (P2)

Como Coordinador de Despacho, necesito registrar la carga física de los paquetes al vehículo y confirmar el despacho, para que el `Módulo de Gestión de Rutas` pueda iniciar el recorrido.

**Why this priority**: Es el último control del módulo sobre el paquete antes de que el `Módulo de Gestión de Rutas` tome la custodia. Sin esta operación no se garantiza la cadena de custodia ni se puede emitir el estado `En Tránsito`.

**Independent Test**: Seleccionar un paquete en `Clasificado` con ruta asignada, ejecutar el flujo de carga completo y verificar que: el estado pase por `En Carga` → `Listo para Despacho`, el registro de carga quede guardado con fecha/hora y responsable, y que al confirmar el vehículo completo se envíe la notificación `vehiculo_listo` al `Módulo de Gestión de Rutas`.

**Acceptance Scenarios**:

1. **Carga exitosa del paquete**
   - **Given** el paquete está en `Clasificado` con ruta y transportador asignados.
   - **When** el coordinador escanea el UUID al recogerlo de la zona de almacenamiento.
   - **Then** el sistema cambia el estado a `En Carga` y registra fecha/hora de inicio e identificador del coordinador.

2. **Confirmación de ubicación en vehículo**
   - **Given** el coordinador verificó que vehículo, zona de destino y ruta son correctos.
   - **When** confirma la ubicación del paquete en el vehículo.
   - **Then** el sistema cambia el estado a `Listo para Despacho` y registra el timestamp de carga completada.

3. **Despacho del vehículo**
   - **Given** todos los paquetes de la ruta están en `Listo para Despacho`.
   - **When** el coordinador confirma que el vehículo está listo para salir.
   - **Then** el sistema envía la notificación `vehiculo_listo` al `Módulo de Gestión de Rutas` con los identificadores de ruta, lista de paquetes y timestamp de despacho.

4. **Bloqueo por estado no permitido**
   - **Given** el paquete está en un estado diferente a `Clasificado` (por ejemplo `En Clasificación` o `Dañado`).
   - **When** el coordinador intenta iniciar la carga.
   - **Then** el sistema bloquea la operación con un mensaje indicando el estado actual y el motivo del bloqueo.

### Edge Cases

- **Dos coordinadores en el mismo paquete**: solo se registra la primera operación (FIFO). La segunda recibe un mensaje con el nombre del coordinador que ya lo está gestionando.
- **Discrepancia de ruta o zona detectada durante la carga**: el coordinador debe escalar al Supervisor de Bodega antes de continuar.
- **Daño detectado durante la manipulación**: el estado revierte a `Clasificado` y el coordinador escala al Controlador de Novedades.
- **Sesión expirada antes de confirmar**: no se registra operación parcial. El paquete permanece en `En Carga`; se genera alerta al Supervisor si supera 60 minutos en ese estado.
- **M2 no responde a `vehiculo_listo`**: el sistema encola la notificación para reintento automático y genera una alerta visible al coordinador.

---

## Requirements

### Functional Requirements

- **FR-001**: Permitir iniciar la carga únicamente sobre paquetes en estado `Clasificado` con ruta y transportador asignados.
- **FR-002**: Cambiar el estado a `En Carga` al escanear el UUID para recoger el paquete, registrando fecha/hora e identificador del coordinador.
- **FR-003**: Cambiar el estado a `Listo para Despacho` al confirmar la ubicación en el vehículo, registrando el timestamp de carga completada.
- **FR-004**: Impedir operaciones duplicadas sobre el mismo paquete (control FIFO).
- **FR-005**: Permitir al coordinador visualizar todos los paquetes de una ruta con su estado de carga.
- **FR-006**: Permitir confirmar el despacho del vehículo cuando todos (o un subconjunto aprobado) de los paquetes de la ruta están en `Listo para Despacho`.
- **FR-007**: Enviar la notificación `vehiculo_listo` al `Módulo de Gestión de Rutas` con: identificador de ruta, identificador del coordinador, lista de UUIDs cargados y timestamp de despacho.
- **FR-008**: Generar alerta al Supervisor de Bodega si un paquete permanece en `En Carga` por más de 60 minutos sin actividad.

### Key Entities

- **Registro de Carga**: UUID del paquete, ruta, coordinador responsable, timestamps de inicio y fin de carga.
- **Registro de Despacho de Vehículo**: identificador de ruta, coordinador, lista de paquetes cargados, timestamp de despacho.

---

## Success Criteria

- **SC-001**: El 100% de los paquetes gestionados quedan registrados correctamente con fecha/hora y responsable.
- **SC-002**: El 100% de los intentos de carga sobre paquetes en estados no permitidos son bloqueados.
- **SC-003**: El tiempo de escaneo y confirmación de carga no supera los 3 minutos en el percentil 95.
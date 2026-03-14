# Feature Specification: Solicitar Ruta de Paquete (MOD1-UC-003)

**Created**: 2026-02-28

## User Scenarios & Testing

### User Story 3 — Solicitud de asignación de ruta al Módulo de Gestión de Rutas (P1)

Como Empleado de Envío y Recepción, necesito que el sistema envíe al `Módulo de Gestión de Rutas` la solicitud de ruta inmediatamente tras la admisión y pesaje exitosos, para reservar un vehículo y proveer al cliente (remitente) un tiempo estimado de entrega desde el primer contacto.

**Why this priority**: Sin este evento el `Módulo de Gestión de Rutas` no puede planificar rutas ni seleccionar vehículos. Se emite en estado `Recibido en Sede`, con datos completos. Es disparado automáticamente por la combinación de [Registrar Admisión de Paquete (MOD1-UC-001)](./MOD1-UC-001-Registrar-Admision-De-Paquete.md) y [Procesar Pesaje y Dimensiones (MOD1-UC-002)](./MOD1-UC-002-Procesar-Pesaje-Y-Dimensiones.md).

**Importante**: La comunicación con el `Módulo de Gestión de Rutas` es estrictamente **asíncrona** mediante payloads en formato JSON. Las solicitudes de ruta para un paquete se generan **una sola vez** y no se pueden re-emitir.

**Independent Test**: Completar admisión y pesaje, y verificar en logs que el payload llegó al `Módulo de Gestión de Rutas`, se recibió una respuesta con identificador de ruta.

**Acceptance Scenarios**:

1. **Solicitud exitosa con asignación de ruta**
   - **Given** el paquete está en `Recibido en Sede` con todos los datos completos.
   - **When** el caso de uso es invocado automáticamente tras [Registrar Admisión de Paquete (MOD1-UC-001)](./MOD1-UC-001-Registrar-Admision-De-Paquete.md) + [Procesar Pesaje y Dimensiones (MOD1-UC-002)](./MOD1-UC-002-Procesar-Pesaje-Y-Dimensiones.md).
   - **Then** el sistema envía el payload JSON de forma asíncrona, recibe un `ID de ruta` en formato JSON del módulo de rutas y lo almacena en el paquete.

2. **M2 no responde**
   - **Given** el `Módulo de Gestión de Rutas` no responde dentro del tiempo configurado.
   - **When** el sistema detecta el timeout.
   - **Then** encola el evento para reintento automático y el cliente recibe la leyenda `Fecha de entrega sujeta a confirmación`.

### Edge Cases

- **¿Qué sucede si el Módulo de Gestión de Rutas rechaza el payload por datos incompletos o inválidos?** El sistema registra el rechazo con el detalle del error devuelto por M2 y notifica al empleado responsable para que revise y corrija los datos del paquete antes de un nuevo intento.
- **¿Qué pasa si el sistema intenta generar una segunda solicitud de ruta para un paquete que ya tiene un `ID de ruta` asignado?** El sistema detecta que la solicitud ya fue emitida y bloquea el reenvío, dado que las solicitudes de ruta se generan **una sola vez** y no se pueden re-emitir.

---

## Requirements

### Functional Requirements

- **FR-001**: La comunicación con el `Módulo de Gestión de Rutas` debe ser estrictamente **asíncrona** mediante payloads en formato JSON.
- **FR-002**: Construir y enviar el payload JSON al `Módulo de Gestión de Rutas` con: UUID del paquete, peso, volumen, tipo de mercancía, dirección de destino, coordenadas GPS.
- **FR-003**: El módulo de rutas devolverá un `ID de ruta` en formato JSON que debe ser almacenado en el paquete.
- **FR-004**: Registrar cada intento con el payload, timestamp, resultado (asignada / pendiente) e ID de ruta recibido.
- **FR-005**: Encolar el evento para reintento automático si M2 no responde, sin bloquear el flujo del paquete.

### Key Entities

- **Solicitud de Ruta**: payload JSON enviado, timestamp, resultado de la respuesta (asignada / pendiente), ID de ruta recibido del módulo de rutas, número de intento.

---

## Success Criteria

- **SC-001**: El 100% de los paquetes con admisión exitosa generan un intento de solicitud asíncrona de forma inmediata.
- **SC-002**: El 0% de las solicitudes se envían con campos faltantes en el payload JSON.
- **SC-003**: El 100% de las solicitudes exitosas reciben un `ID de ruta` del módulo de rutas y lo almacenan en el paquete.
- **SC-004**: El 100% de los clientes (remitentes) son informados del tiempo de entrega de 7 días hábiles cuando la ruta es asignada exitosamente.
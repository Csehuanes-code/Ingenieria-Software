# Feature Specification: Solicitar Ruta de Paquete (MOD1-UC-003)

**Created**: 2026-02-28

## User Scenarios & Testing

### User Story 1 — Solicitud de asignación de ruta al Módulo de Gestión de Rutas (P1)

Como Sistema, necesito enviar al `Módulo de Gestión de Rutas` la solicitud de ruta inmediatamente tras la admisión y pesaje exitosos, para reservar un vehículo y proveer al cliente un tiempo estimado de entrega desde el primer contacto.

**Why this priority**: Sin este evento el `Módulo de Gestión de Rutas` no puede planificar rutas ni seleccionar vehículos. Se emite en estado `Recibido en Sede`, con GPS resuelto y datos completos. Es disparado automáticamente por la combinación de [MOD1-UC-001](./MOD1-UC-001-Registrar-Admision-De-Paquete.md) y [MOD1-UC-002](./MOD1-UC-002-Procesar-Pesaje-Y-Dimensiones.md).

**Independent Test**: Completar admisión y pesaje, y verificar en logs que el payload llegó al `Módulo de Gestión de Rutas`, se recibió una respuesta con identificador de ruta, transportador y fecha estimada, y que esa información se mostró al cliente.

**Acceptance Scenarios**:

1. **Solicitud exitosa**
   - **Given** el paquete está en `Recibido en Sede` con GPS resuelto y todos los datos completos.
   - **When** el caso de uso es invocado automáticamente tras [MOD1-UC-001](./MOD1-UC-001-Registrar-Admision-De-Paquete.md) + [MOD1-UC-002](./MOD1-UC-002-Procesar-Pesaje-Y-Dimensiones.md).
   - **Then** el sistema envía el payload, persiste la respuesta (ruta, transportador y fecha estimada) y muestra el tiempo estimado al cliente.

2. **M2 rechaza por cobertura no disponible**
   - **Given** el `Módulo de Gestión de Rutas` responde que el destino está fuera de cobertura.
   - **When** el sistema recibe el rechazo.
   - **Then** el paquete pasa a `Excepción de Ruta`, se notifica al Supervisor de Admisión y se habilita la corrección de dirección para reintentar.

3. **M2 no responde**
   - **Given** el `Módulo de Gestión de Rutas` no responde dentro del tiempo configurado.
   - **When** el sistema detecta el timeout.
   - **Then** encola el evento para reintento automático y el cliente recibe la leyenda `Fecha de entrega sujeta a confirmación`.

### Escenarios donde se permite re-emitir la solicitud

La solicitud se emite idealmente una vez por ciclo de vida. Las excepciones son:

| Escenario | Quién gestiona |
|---|---|
| Rechazo por cobertura — corrección de dirección | Supervisor de Admisión |
| Diferencia significativa de datos físicos en bodega, corregida y aprobada | Supervisor de Bodega |
| Cambio de dirección autorizado antes del almacenaje | Supervisor de Admisión |

### Condiciones de Red

| Condición | Comportamiento |
|---|---|
| Normal (RTT < 200 ms) | Respuesta esperada en < 5 s |
| Degradada (RTT 200–1.000 ms) | Hasta 15 s con reintentos en backoff exponencial |
| No disponible (timeout > 5 s) | Evento encolado; reintentos cada 30 s, máximo 10 intentos; si falla, alerta al Administrador |

### Edge Cases

- **GPS pendiente**: el sistema bloquea el envío hasta que las coordenadas estén resueltas.
- **Datos físicos corregidos en bodega**: si la corrección afecta el payload, se re-emite la solicitud y se notifica al remitente con los nuevos tiempos estimados.
- **Novedad tras emitir fecha estimada**: se genera una notificación interna para que atención al cliente informe al destinatario sobre la anulación o cambio de fecha.

---

## Requirements

### Functional Requirements

- **FR-001**: Construir y enviar el payload al `Módulo de Gestión de Rutas` con: UUID del paquete, peso, volumen, tipo de mercancía, dirección de destino, coordenadas GPS, método de pago y valor declarado.
- **FR-002**: Registrar cada intento con el payload, timestamp, resultado (asignada / rechazada / pendiente), identificadores de ruta y transportador, fecha estimada y motivo de rechazo cuando aplique.
- **FR-003**: Mostrar la fecha estimada al cliente al recibirla. Si está pendiente, mostrar la leyenda `Fecha de entrega sujeta a confirmación`.
- **FR-004**: Encolar el evento para reintento automático si M2 no responde, sin bloquear el flujo del paquete.
- **FR-005**: Bloquear el envío si el GPS del paquete está pendiente.
- **FR-006**: Permitir la re-emisión únicamente en los escenarios definidos, con autorización del Supervisor correspondiente.
- **NFR-001**: El ciclo de solicitud y respuesta debe completarse en < 5 s bajo condiciones normales de red.

### Key Entities

- **Solicitud de Ruta**: payload enviado, timestamp, resultado de la respuesta, identificadores de ruta y transportador, fecha estimada, motivo de rechazo, número de intento.

---

## Success Criteria

- **SC-001**: El 100% de los paquetes con admisión exitosa y GPS resuelto generan un intento de solicitud de forma inmediata.
- **SC-002**: El 0% de las solicitudes se envían con campos faltantes respecto al contrato de integración.
- **SC-003**: El ciclo de solicitud y respuesta se completa en < 5 s en el percentil 95 bajo condiciones normales.
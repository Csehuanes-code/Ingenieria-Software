# Feature Specification: Solicitar Ruta de Paquete (MOD1-UC-003)

**Created**: 2026-02-28

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Solicitud oficial de asignación de ruta y vehículo (Priority: P1)

Como Sistema, necesito enviar al `Módulo de Gestión de Rutas` la solicitud de asignación de ruta inmediatamente después de que el pesaje y la admisión sean completados exitosamente, para que el `Módulo de Gestión de Rutas` inicie la planificación del transporte y pueda entregar al cliente, desde el primer contacto, un tiempo estimado de entrega y la confirmación del servicio.

**Why this priority**: Es el primer evento de integración entre el `Módulo de Gestión de Paquetes` y el `Módulo de Gestión de Rutas`. Sin este evento el M2 no puede asignar vehículo, calcular ruta ni confirmar la viabilidad del envío. Se emite cuando el paquete está en estado `Recibido en Sede`, una vez que los campos `peso_kg`, `volumen_m3`, `tipo_mercancia`, `latitud`, `longitud`, `metodo_pago` y `valor_declarado` están disponibles y validados. Es invocado automáticamente por la combinación exitosa de [Registrar Admisión de Paquete (MOD1-UC-001)](./MOD1-UC-001-Registrar-Admision-De-Paquete.md) y [Procesar Pesaje y Dimensiones (MOD1-UC-002)](./MOD1-UC-002-Procesar-Pesaje-Y-Dimensiones.md).

**Independent Test**: Puede probarse completando exitosamente el flujo de admisión y pesaje, y verificando en los logs que el sistema construyó y envió el payload JSON completo al `Módulo de Gestión de Rutas`. El test es exitoso si el sistema recibió `ruta_asignada` con `id_ruta`, `id_transportador` y `fecha_estimada_entrega`, y que la información de tiempo estimado fue mostrada al cliente.

**Acceptance Scenarios**:

1. **Scenario**: Solicitud de ruta exitosa — estado Recibido en Sede
   - **Given** el paquete está en estado `Recibido en Sede` con `gps_estado = Resuelto` y todos los campos del payload completos y válidos, y el `Módulo de Gestión de Rutas` está disponible.
   - **When** este caso de uso es invocado automáticamente tras la confirmación exitosa de UC-001 + UC-002.
   - **Then** el sistema construye el payload según el Contrato de Integración (ver FR-001) y lo envía al `Módulo de Gestión de Rutas`. Al recibir `ruta_asignada`, persiste `id_ruta`, `id_transportador` y `fecha_estimada_entrega`, y muestra el tiempo estimado de entrega al cliente en pantalla.

2. **Scenario**: `Módulo de Gestión de Rutas` rechaza la solicitud por cobertura no disponible
   - **Given** el `Módulo de Gestión de Rutas` responde indicando que la dirección de destino está fuera de las zonas de cobertura configuradas.
   - **When** el sistema recibe la respuesta de rechazo por cobertura.
   - **Then** el sistema asigna el estado `Excepción de Ruta` al paquete, notifica al Supervisor de Admisión con el detalle del rechazo, y habilita la corrección manual de la dirección. Una vez corregida la dirección y resueltas las coordenadas GPS, se puede reintentar la solicitud. Este es uno de los escenarios donde se permite re-emitir el evento.

3. **Scenario**: `Módulo de Gestión de Rutas` no responde — encolamiento para reintento automático
   - **Given** el `Módulo de Gestión de Rutas` no responde dentro del tiempo de espera configurado.
   - **When** el sistema detecta el timeout.
   - **Then** el sistema encola el evento para reintento automático y mantiene el estado `Recibido en Sede`. Genera una alerta al Empleado de Envío y Recepción indicando que la confirmación de ruta está pendiente, y emite el comprobante de admisión con la leyenda `Fecha de entrega sujeta a confirmación`.

---

### Escenarios de Reintento Permitidos

La solicitud de ruta debe emitirse idealmente **una sola vez** por ciclo de vida del paquete. Sin embargo, los siguientes son escenarios donde se permite re-emitir el evento:

| Escenario | Condición para reintento | Actor que lo gestiona |
|---|---|---|
| Rechazo por cobertura no disponible | Corrección de dirección y GPS por parte del empleado | Supervisor de Admisión |
| Diferencia significativa de datos físicos detectada en bodega | Bloqueo del paquete, corrección de datos validada por supervisor | Supervisor de Bodega (ver nota de roles) |
| Cambio de dirección de destino solicitado por el remitente | Antes de la asignación de zona de almacenamiento; requiere autorización | Supervisor de Admisión |

**Nota sobre Roles de Supervisor**: En el contexto del `Módulo de Gestión de Paquetes`, el término "Supervisor" hace referencia al usuario autenticado con nivel de acceso superior al empleado operativo. Según el diagrama de casos de uso del módulo, los actores definidos son: Empleado de Envío y Recepción, Almacenista, Coordinador de Despacho, Controlador de Novedades, Módulo de Gestión de Rutas y Módulo de Gestión de Finanzas. El rol de "Supervisor" corresponde a un usuario con nivel de acceso que abarca las funciones de supervisión operativa dentro de cada área; su definición exacta de permisos pertenece al módulo de gestión de usuarios del sistema, fuera del alcance de MOD1.

---

### Edge Cases

- What happens when el paquete tiene `gps_estado = Pendiente GPS`? El sistema bloquea la emisión del evento. La solicitud queda pendiente hasta que el empleado ingrese coordenadas válidas manualmente y `gps_estado` cambie a `Resuelto`.
- What happens when el paquete tiene `prioridad = Urgente`? El payload incluye el campo `prioridad: "Urgente"`. El `Módulo de Gestión de Rutas` deberá asignar el primer vehículo disponible en la zona y garantizar despacho en el mismo día hábil si la admisión ocurre antes de las 12:00 hrs.
- What happens when el M2 no responde y el paquete pasa a estado de novedad antes de que el reintento automático tenga éxito? El sistema cancela los reintentos pendientes del evento `solicitar_ruta` y registra el evento como `Cancelado por Novedad` en el log de Solicitudes de Ruta.
- What happens when se detecta en bodega una diferencia significativa de datos físicos (peso > 10% del registrado)? El paquete se marca como `Fuera de Tolerancia — Peso`. El flujo queda suspendido hasta que el Supervisor de Bodega corrija y apruebe los nuevos datos. Una vez aprobados, si la diferencia afecta el payload (`peso_kg`, `volumen_m3`), se re-emite la solicitud de ruta con los datos corregidos.

### Condiciones de Red — Tiempos de Respuesta Esperados

| Condición de red | Latencia esperada | Comportamiento del sistema |
|---|---|---|
| **Normal** | Latencia RTT < 200 ms; ancho de banda ≥ 10 Mbps; sin pérdida de paquetes | Ciclo de solicitud y respuesta completado en < 5 segundos. |
| **Degradada** | Latencia RTT entre 200 ms y 1.000 ms; pérdida de paquetes < 5% | Ciclo puede tardar entre 5 y 15 segundos. El sistema aplica reintentos con backoff exponencial (1s, 2s, 4s) hasta 3 intentos antes de encolar. |
| **No disponible / timeout** | Sin respuesta en > 5 segundos o error de conexión | El sistema encola el evento en la cola de mensajes (RabbitMQ/Kafka) para reintento automático cada 30 segundos hasta un máximo de 10 intentos. Si los 10 intentos fallan, el evento queda en estado `Fallo Persistente` y se genera una alerta al Administrador del Sistema. |

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST construir y enviar al `Módulo de Gestión de Rutas` el payload JSON con los siguientes campos obligatorios según el Contrato de Integración:
  - `evento`: `"solicitar_ruta"`
  - `fecha_hora`: timestamp UTC de la solicitud
  - `paquete.id`: UUID del paquete
  - `paquete.peso_kg`: peso real registrado
  - `paquete.volumen_m3`: volumen calculado
  - `paquete.tipo_mercancia`: `"Estándar"` | `"Frágil"` | `"Peligroso"`
  - `paquete.direccion_destino`: texto completo de la dirección
  - `paquete.latitud`: coordenada GPS de destino
  - `paquete.longitud`: coordenada GPS de destino
  - `paquete.metodo_pago`: `"prepago"` | `"contra_entrega"`
  - `paquete.valor_declarado`: monto declarado en moneda local
  - `paquete.prioridad`: `"Estándar"` | `"Urgente"` (opcional; se incluye solo si `prioridad = Urgente`)

  Si algún campo está ausente o inválido, el sistema no emite la solicitud y notifica el campo faltante.

- **FR-002**: System MUST registrar en la entidad `Solicitud de Ruta` el estado de cada intento: `payload_enviado`, `timestamp_envio`, `estado_respuesta` (`asignada` | `rechazada` | `pendiente` | `cancelada`), `id_ruta`, `id_transportador`, `fecha_estimada_entrega`, `motivo_rechazo`.
- **FR-003**: System MUST mostrar la `fecha_estimada_entrega` recibida del `Módulo de Gestión de Rutas` al cliente (en la pantalla de confirmación de admisión y en el comprobante impreso). Si la confirmación está pendiente, el comprobante debe incluir la leyenda `Fecha de entrega sujeta a confirmación`.
- **FR-004**: System MUST encolar el evento para reintento automático si el `Módulo de Gestión de Rutas` no responde dentro de 5 segundos, aplicando los reintentos y timeouts según la tabla de condiciones de red (ver sección anterior).
- **FR-005**: System MUST bloquear la emisión si `gps_estado = Pendiente GPS`.
- **FR-006**: System MUST permitir la re-emisión de la solicitud únicamente en los escenarios listados en la tabla de reintentos permitidos, requiriendo autorización del rol Supervisor correspondiente en cada caso.
- **NFR-001**: System MUST completar el ciclo de solicitud y respuesta en < 5 s bajo condiciones de red normales (RTT < 200 ms).
- **NFR-002**: System MUST iniciar el primer reintento automático dentro de los 30 segundos siguientes al timeout inicial, usando backoff exponencial para intentos sucesivos.

### Key Entities *(include if feature involves data)*

- **Solicitud de Ruta**: `id` (PK), `uuid_paquete` (FK), `payload_enviado` (JSON), `timestamp_envio` (UTC), `estado_respuesta`, `id_ruta` (nullable), `id_transportador` (nullable), `fecha_estimada_entrega` (nullable), `motivo_rechazo` (nullable), `numero_intento` (entero).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los paquetes con registro y pesaje exitosos y `gps_estado = Resuelto` deben generar un intento de solicitud de ruta de forma inmediata.
- **SC-002**: El 0% de las solicitudes deben enviarse con campos faltantes o inválidos respecto al Contrato de Integración.
- **SC-003**: El ciclo de solicitud y respuesta debe completarse en < 5 s en el percentil 95 bajo condiciones de red normales.
# Feature Specification: Solicitar Ruta de Paquete (MOD1-UC-003)

**Created**: 2026-02-28

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Solicitud oficial de asignación de ruta y vehículo (Priority: P1)

Como Sistema (Módulo 1), necesito enviar al Módulo 2 la solicitud oficial y definitiva de asignación de ruta inmediatamente tras el registro y pesaje del paquete, para reservar espacio en la flota y proveer al cliente una fecha estimada de despacho desde el primer contacto.

**Why this priority**: Es la solicitud formal y única de asignación de ruta por ciclo de vida normal del paquete. Sin este evento el Módulo 2 no puede planificar rutas ni seleccionar vehículos. Es invocado automáticamente por [Registrar Admisión de Paquete(MOD1-UC-001)](MOD1-UC-001-Registrar-Admision-De-Paquete.md).

**Independent Test**: Puede probarse completando exitosamente un registro de admisión y verificando en los logs que el sistema envió el JSON completo (según el Contrato de Integración) al Módulo 2, recibió confirmación con ventana de tiempo de despacho y que dicha información quedó impresa en el comprobante del cliente.

**Acceptance Scenarios**:

1. **Scenario**: Solicitud de ruta oficial exitosa con emisión de comprobante
   - **Given** el paquete ha sido registrado y pesado exitosamente (estado `Recibido en Sede`), posee coordenadas GPS válidas y el Módulo 2 está disponible.
   - **When** el caso de uso es invocado automáticamente por [Registrar Admisión de Paquete(MOD1-UC-001)](MOD1-UC-001-Registrar-Admision-De-Paquete.md).
   - **Then** el sistema construye y envía el payload JSON completo al Módulo 2, recibe la confirmación con la ventana de tiempo de despacho asignada, muestra la información en pantalla y la plasma en el comprobante impreso entregado al cliente.

2. **Scenario**: Módulo 2 rechaza la solicitud por cobertura no disponible — Excepción de Ruta
   - **Given** el Módulo 2 responde indicando que la dirección de destino está fuera de las zonas de cobertura configuradas.
   - **When** el sistema recibe la respuesta de rechazo por cobertura.
   - **Then** el sistema asigna el estado `Excepción de Ruta` al paquete, notifica al supervisor con el detalle del rechazo y habilita la corrección manual de la dirección para un reintento. Este es el único escenario donde se permite una segunda solicitud de ruta en el ciclo normal.

3. **Scenario**: Módulo 2 no responde — encolamiento para reintento automático
   - **Given** el Módulo 2 no responde dentro del tiempo de espera configurado.
   - **When** el sistema detecta el timeout de la solicitud.
   - **Then** el sistema encola el evento para reintento automático (mediante cola de mensajes), emite el comprobante con la leyenda `Fecha de despacho sujeta a confirmación` y el paquete continúa su ciclo operativo sin bloqueo.

---

### Edge Cases

- What happens when el Módulo 2 rechaza la solicitud por cobertura no disponible? Es el único escenario que permite reintentar la solicitud. El paquete entra en estado `Excepción de Ruta`. Un supervisor corrige la dirección de destino y reintenta la solicitud manualmente.
- What happens when el Módulo 2 no responde en tiempo? El evento de solicitud se encola en la cola de mensajes (RabbitMQ/Kafka) para reintento automático. El comprobante se emite con la leyenda `Sujeto a confirmación`, sin bloquear el flujo del paquete.
- What happens when el paquete no tiene coordenadas GPS válidas al momento de invocar este caso de uso? El sistema no envía la solicitud. El paquete queda en estado `Pendiente GPS` y la solicitud de ruta permanece bloqueada hasta que las coordenadas sean resueltas.
- What happens when el paquete es marcado como `Urgente`? El sistema incluye la marca de prioridad en el payload enviado al Módulo 2, solicitando explícitamente el primer espacio disponible en la flota de la zona, y refleja la fecha prioritaria resultante en el comprobante.
- How does system handle si el paquete pasa a estado `Novedad` poco después de haber entregado el comprobante físico al cliente? El sistema genera una notificación interna al área de atención al cliente para que informen al destinatario la anulación o actualización de la fecha de despacho comprometida.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST construir y enviar al Módulo 2 el payload JSON oficial definido en el Contrato de Integración, incluyendo: UUID, peso_kg, volumen_m3, tipo_mercancia, direccion_destino, latitud, longitud, metodo_pago y valor_declarado.
- **FR-002**: System MUST garantizar que esta solicitud se emita una única vez por ciclo de vida normal del paquete. El único reintento permitido es ante rechazo por cobertura (estado `Excepción de Ruta`) con corrección supervisada de la dirección.
- **FR-003**: System MUST calcular y mostrar el tiempo estimado de despacho a partir de la respuesta del Módulo 2, plasmándolo en el comprobante de recepción entregado al cliente.
- **FR-004**: System MUST soportar la asignación de prioridad `Urgente` en el payload para que el Módulo 2 asigne el primer espacio disponible en flota.
- **FR-005**: System MUST encolar el evento de solicitud para reintento automático si el Módulo 2 no responde, sin bloquear el flujo principal del paquete.
- **FR-006**: System MUST bloquear la ejecución de este caso de uso si el paquete no tiene coordenadas GPS válidas, manteniendo el estado `Pendiente GPS` hasta su resolución.
- **NFR-001**: System MUST garantizar que el 95% de las solicitudes con respuesta exitosa del Módulo 2 retornen una fecha estimada con un margen de error no mayor a ±1 día operativo.
- **NFR-002**: System MUST completar el ciclo de solicitud y recepción de respuesta del Módulo 2 en menos de 5 segundos bajo condiciones normales de red.

### Key Entities *(include if feature involves data)*

- **Solicitud de Ruta**: Entidad que representa el evento oficial enviado al Módulo 2, incluyendo el payload completo y el estado de respuesta recibido (asignada, rechazada, pendiente).
- **Zona de Destino**: Clasificación geográfica derivada de las coordenadas GPS utilizada para identificar la ruta y flota correspondientes en el Módulo 2.
- **Comprobante de Recepción**: Documento físico entregado al cliente que incluye el UUID del paquete y la fecha estimada de despacho.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los registros de admisión exitosos deben generar una solicitud de ruta al Módulo 2 de forma automática e inmediata.
- **SC-002**: El 95% de las solicitudes con respuesta exitosa del Módulo 2 deben retornar una fecha estimada con un margen de error ≤ 1 día operativo.
- **SC-003**: El 100% de los comprobantes emitidos deben incluir la fecha estimada de despacho o explícitamente la leyenda `Sujeto a confirmación`.
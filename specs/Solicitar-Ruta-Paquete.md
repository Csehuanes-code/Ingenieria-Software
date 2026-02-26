# Feature Specification: Solicitar Ruta de Paquete Preliminar (MOD1-UC-008)

**Created**: 2026-02-26

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Proyección informativa de entrega (Priority: P2-Alta)

Como Empleado de Envío y Recepción, necesito consultar preliminarmente al Módulo 2 una estimación de ruta para poder imprimir en el comprobante del cliente su fecha estimada de despacho.

**Why this priority**: Permite mejorar el nivel de servicio informando al cliente la fecha estimada desde el primer contacto, tratándose de una solicitud informativa que no asigna ni bloquea capacidad de flota.

**Independent Test**: Puede probarse enviando una consulta preliminar de solo lectura al Módulo 2 con coordenadas válidas y validando que el tiempo estimado retornado se imprima correctamente en el recibo.

**Acceptance Scenarios**:

1. **Scenario**: Emisión de comprobante con estimación
   - **Given**: El paquete ha sido registrado en estado 'Recibido en Sede', posee GPS válido y el Módulo 2 se encuentra disponible para responder.
   - **When**: El empleado confirma la solicitud de estimación al término del registro principal.
   - **Then**: El sistema envía los datos de zona al Módulo 2, recibe la ventana de tiempo de despacho, la muestra en pantalla y la plasma en el comprobante impreso.

### Edge Cases

- What happens when: el Módulo 2 informa que no hay rutas activas o programadas para el destino en el corto plazo? El sistema muestra el aviso "Fecha de despacho por confirmar" y el comprobante se emite vacío en ese campo.
- What happens when: el empleado activa la prioridad 'Urgente' sobre el paquete? El sistema solicita explícitamente la fecha del primer espacio disponible en la flota y actualiza el comprobante con la nueva fecha y un indicador.
- How does system handle: si el paquete pasa a estado 'Novedad' poco después de haber entregado el comprobante físico al cliente? El sistema genera una notificación paralela al área de atención al cliente para que informen la anulación o actualización de dicha fecha.
- What happens when: el Módulo 2 no responde la consulta de estimación a tiempo? El sistema no bloquea el flujo principal de registro; asume "Por confirmar" en el comprobante y el paquete continúa su ciclo operativo normalmente.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST: filtrar y consultar las rutas disponibles en el Módulo 2 basándose exclusivamente en la zona geográfica del paquete.
- **FR-002**: System MUST: calcular internamente y mostrar el tiempo estimado de despacho según la respuesta de disponibilidad obtenida.
- **FR-003**: System MUST: incluir y asegurar la impresión de la fecha estimada de despacho en el formato del comprobante de recepción.
- **FR-004**: System MUST: soportar lógicamente la asignación de prioridad 'Urgente' para agilizar la búsqueda del primer espacio disponible en flota.
- **NFR-001**: System MUST: garantizar que el 95% de las fechas estimadas proyectadas tengan un margen de error no mayor a $\pm1$ día operativo.
- **NFR-002**: System MUST: completar y obtener la respuesta de estimación de ruta en un lapso inferior a 5 segundos desde la ejecución de la solicitud.

### Key Entities *(include if feature involves data)*

- **Ruta Preliminar**: Entidad conceptual de proyección informativa que no ejecuta reservas definitivas ni registros en el Módulo 2.
- **Zona de Destino**: Variable y clasificación geográfica derivada del GPS utilizada para la consulta de disponibilidad.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 95% de las estimaciones preliminares consultadas deben retornar una fecha estimada con un margen de error $\le1$ día.
- **SC-002**: El 100% de los comprobantes emitidos en sede deben incluir la fecha proyectada o explícitamente el mensaje de validación "Por confirmar".
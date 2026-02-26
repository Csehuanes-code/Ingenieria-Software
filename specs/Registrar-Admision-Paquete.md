# Feature Specification: Registrar Admisión de Paquete (MOD1-UC-001)

**Created**: 2026-02-26

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Ingreso inicial y generación de identidad (Priority: P1 Crítica)

Como Empleado de Envío y Recepción, necesito registrar los datos de un paquete nuevo para generar su identidad única e iniciar su ciclo de vida en el sistema.

**Why this priority**: Es el objetivo principal para formalizar el ingreso; sin este registro, el paquete no existe para ningún otro módulo del sistema.

**Independent Test**: Puede ser probado independientemente verificando que, al completar el formulario, se genera un UUID único, la dirección se convierte a coordenadas GPS y el estado inicial es 'Recibido en Sede'.

**Acceptance Scenarios**:

1. **Scenario**: Registro exitoso de paquete
   - **Given**: El empleado está autenticado en el sistema y el paquete ha sido verificado físicamente.
   - **When**: El empleado completa los datos, el sistema valida las coordenadas GPS y el empleado confirma el registro.
   - **Then**: El sistema genera un UUID único, registra fecha/hora y sede automáticamente, emite la etiqueta física y actualiza el estado a 'Recibido en Sede'.

### Edge Cases

- What happens when: falla el servicio de geolocalización (timeout > 5 segundos)? El sistema muestra un aviso al empleado, permitiendo ingresar coordenadas manualmente o guardar la dirección con estado 'Pendiente GPS' para reintentos automáticos.
- What happens when: el método de pago no es soportado en la sede? El sistema muestra un mensaje con los métodos disponibles, y el empleado debe ajustar el registro o cancelarlo.
- How does system handle: la generación de un UUID duplicado por colisión? El sistema valida la unicidad en la base de datos y regenera el UUID hasta obtener uno único; nunca se persiste un duplicado.
- What happens when: la dirección de destino está en una zona sin cobertura de GPS? El sistema registra la dirección en texto plano, marca el campo GPS como 'Sin Resolución' y bloquea la transición a clasificación hasta que un operador lo resuelva manualmente.
- What happens when: el empleado intenta registrar un paquete con valor declarado en cero? El sistema lo permite solo si la mercancía es 'Estándar'; para mercancías 'Frágil' o 'Peligroso', exige un valor mayor a cero por cláusulas de seguro.
- How does system handle: si el cliente retira el paquete antes de confirmar el registro? El registro queda en estado 'Borrador' por 30 minutos antes de ser eliminado automáticamente.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST: generar un UUID (v4) único e irrepetible para cada nuevo paquete registrado.
- **FR-002**: System MUST: registrar automáticamente la fecha/hora de ingreso (timestamp UTC) y la sede de origen sin intervención del empleado.
- **FR-003**: System MUST: capturar y persistir el valor declarado y el método de pago (prepago o contra entrega).
- **FR-004**: System MUST: convertir la dirección de destino en coordenadas GPS (latitud/longitud) utilizando un servicio de geocodificación.
- **FR-005**: System MUST: validar que los campos obligatorios estén completos antes de confirmar el registro.
- **FR-006**: System MUST: generar y emitir una etiqueta física (formato PDF/ZPL) con el UUID en código de barras o QR al confirmar el registro.
- **FR-007**: System MUST: impedir la edición del UUID, la fecha de ingreso y la sede de origen una vez confirmado el registro.
- **FR-008**: System MUST: permitir el ingreso de coordenadas GPS manuales como contingencia ante fallo del servicio de geocodificación.
- **NFR-001**: System MUST: garantizar que el tiempo de confirmación del registro completo no supere los 2 minutos por paquete en condiciones normales.
- **NFR-002**: System MUST: mantener el módulo de admisión disponible el 99.5% del tiempo en horario operativo (6:00 AM - 8:00 PM).
- **NFR-003**: System MUST: permitir que solo usuarios con rol 'Empleado de Envío y Recepción' o superior creen registros de admisión.
- **NFR-004**: System MUST: vincular cada registro al ID del empleado que lo creó, sin posibilidad de modificación posterior.

### Key Entities *(include if feature involves data)*

- **Paquete**: Entidad central que contiene UUID, estados, fechas de registro y atributos físicos, comerciales y geográficos.
- **Datos Comerciales**: Subentidad del Paquete que incluye valor declarado, costo de envío y método de pago.
- **Datos Geográficos**: Subentidad del Paquete que incluye dirección exacta, latitud y longitud.
- **Sede**: Entidad que representa el punto de admisión y se vincula automáticamente al paquete.
- **Historial de Estados**: Registro cronológico de todas las transiciones de estado del paquete.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los paquetes registrados deben tener un UUID único, validado mediante consulta en base de datos.
- **SC-002**: El tiempo total de registro de admisión no debe superar los 2 minutos por paquete en el percentil 95.
- **SC-003**: El 100% de los registros deben tener fecha/hora de ingreso y sede de origen capturadas automáticamente.
- **SC-004**: El 95% de las direcciones registradas deben resolverse en coordenadas GPS válidas en tiempo real.
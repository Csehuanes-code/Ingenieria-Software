# Feature Specification: Registrar Admisión de Paquete (MOD1-UC-001)

**Created**: 2026-02-28

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Ingreso inicial y orquestación del ciclo de vida (Priority: P1)

Como Empleado de Envío y Recepción, necesito registrar los datos completos de un paquete nuevo —incluyendo remitente, destinatario, dirección de destino y atributos físicos— para generar su identidad única en el sistema y dejarlo correctamente ingresado con toda la información necesaria para su posterior clasificación y despacho.

**Why this priority**: Es el punto de entrada al sistema. Sin este registro el paquete no existe para ningún otro módulo. Orquesta la captura de datos del cliente y el pesaje obligatorio. La solicitud de ruta al `Módulo de Gestión de Rutas` no ocurre aquí, sino más adelante cuando se han registrado todos los datos obligatorios del paquete.

**Independent Test**: Puede probarse completando el formulario con datos válidos de remitente, destinatario, dirección y tipo de mercancía, ejecutando el pesaje incluido y confirmando el registro. El test es exitoso si el sistema genera un UUID único, emite la etiqueta física y el paquete queda en estado `Recibido en Sede` sin haber enviado ningún evento al `Módulo de Gestión de Rutas`.

**Acceptance Scenarios**:

1. **Scenario**: Registro exitoso con pesaje obligatorio incluido
   - **Given** el empleado está autenticado y el cliente entrega el paquete físicamente con sus datos de contacto.
   - **When** el empleado ingresa los datos del remitente y destinatario, el sistema valida las coordenadas GPS de la dirección, se ejecuta obligatoriamente el caso de uso [Procesar Pesaje y Dimensiones (MOD1-UC-002)](./MOD1-UC-002-Procesar-Pesaje-Y-Dimensiones.md), y el empleado confirma el registro.
   - **Then** el sistema genera un UUID único, registra la fecha/hora y sede automáticamente, actualiza el estado a `Recibido en Sede` y emite la etiqueta física del paquete.

2. **Scenario**: Registro con fallo de geolocalización — estado Pendiente GPS
   - **Given** el empleado completa todos los campos del formulario pero el servicio de geocodificación no responde (timeout > 5 segundos).
   - **When** el sistema detecta el fallo del servicio GPS.
   - **Then** el sistema permite guardar la dirección en texto plano, asigna el estado `Pendiente GPS` al campo geográfico del paquete y notifica al empleado. El paquete podrá avanzar en el flujo hasta que las coordenadas sean resueltas, momento en el que el bloqueo se levanta automáticamente.

---

### Edge Cases

- What happens when el servicio de geocodificación falla (timeout > 5 segundos)? El sistema muestra un aviso al empleado. Se permite ingresar coordenadas manualmente o guardar la dirección en texto plano con estado `Pendiente GPS`. El paquete queda bloqueado para avanzar a clasificación hasta que las coordenadas sean válidas.
- What happens when se intenta confirmar el registro sin haber completado el pesaje? El sistema bloquea la confirmación y muestra un mensaje indicando que el caso de uso [Procesar Pesaje y Dimensiones (MOD1-UC-002)](./MOD1-UC-002-Procesar-Pesaje-Y-Dimensiones.md) es obligatorio antes de poder guardar el registro.
- How does system handle un intento de registrar un paquete con método de pago no soportado en la sede actual? El sistema valida el método contra la configuración de la sede y muestra los métodos válidos disponibles, impidiendo el guardado hasta seleccionar uno compatible.
- What happens when el empleado cancela el formulario después de que ya se generó un UUID preliminar? El registro queda en estado `Borrador` por 30 minutos y es eliminado automáticamente si no se confirma. No se emite ningún evento externo.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST capturar y convertir la dirección de destino ingresada en coordenadas geográficas (latitud y longitud) utilizando un servicio de geocodificación externo.
- **FR-002**: System MUST generar un UUID (v4) único e irrepetible para cada paquete registrado y validar su unicidad contra la base de datos antes de persistir.
- **FR-003**: System MUST registrar automáticamente la fecha/hora de ingreso (timestamp UTC) y la sede de origen, sin intervención del empleado.
- **FR-004**: System MUST capturar el Valor Declarado y el Método de Pago (prepago o contra entrega) como campos obligatorios.
- **FR-005**: System MUST validar que todos los campos obligatorios estén completos antes de permitir la confirmación del registro.
- **FR-006**: System MUST generar y emitir una etiqueta física del paquete (ZPL con UUID en código QR o barras) al confirmar el registro.
- **FR-007**: System MUST impedir la edición del UUID, la fecha de ingreso y la sede de origen una vez confirmado el registro.
- **FR-008**: System MUST permitir el ingreso manual de coordenadas GPS como contingencia ante fallo del servicio de geocodificación.
- **FR-009**: System MUST capturar los datos del Remitente (Documento, Nombre, Teléfono) y del Destinatario (Documento, Nombre, Teléfono, Email) como campos obligatorios del registro.
- **FR-010**: System MUST invocar el caso de uso [Procesar Pesaje y Dimensiones (MOD1-UC-002)](./MOD1-UC-002-Procesar-Pesaje-Y-Dimensiones.md) como parte obligatoria e inseparable del flujo de registro, impidiendo la confirmación si el pesaje no fue completado.

### Key Entities *(include if feature involves data)*

- **Paquete**: Entidad central que contiene UUID, estados del ciclo de vida, fechas de registro y atributos físicos, comerciales y geográficos.
- **Remitente**: Entidad con Documento, Nombre y Teléfono de quien origina el envío.
- **Destinatario**: Entidad con Documento, Nombre, Teléfono y Email de quien recibe el paquete.
- **Sede**: Entidad que representa el punto de admisión, vinculada automáticamente al paquete en el registro.
- **Historial de Estados**: Registro cronológico e inmutable de todas las transiciones de estado del paquete.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los paquetes registrados deben tener un UUID único, con datos de remitente y destinatario completos capturados.
- **SC-002**: El 95% de las direcciones registradas deben resolverse en coordenadas GPS válidas en tiempo real durante el registro.
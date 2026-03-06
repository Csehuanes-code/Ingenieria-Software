# Feature Specification: Registrar Admisión de Paquete (MOD1-UC-001)

**Created**: 2026-02-28

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Ingreso inicial y orquestación del ciclo de vida (Priority: P1)

Como Empleado de Envío y Recepción, necesito registrar los datos completos de un paquete nuevo —incluyendo los datos de identificación del remitente, los datos de contacto y entrega del destinatario, y los atributos físicos del paquete (peso, dimensiones y tipo de mercancía)— para generar la identidad única del paquete en el sistema y disparar automáticamente la solicitud de ruta al `Módulo de Gestión de Rutas` una vez que el pesaje sea completado exitosamente.

**Why this priority**: Es el punto de entrada al sistema. Sin este registro el paquete no existe para ningún otro módulo. Una vez confirmado el pesaje (MOD1-UC-002), el paquete dispone de todos los campos requeridos por el Contrato de Integración (`peso_kg`, `volumen_m3`, `tipo_mercancia`, `latitud`, `longitud`, `metodo_pago`, `valor_declarado`) y por lo tanto la solicitud de ruta puede y debe emitirse de forma inmediata, sin necesidad de esperar a etapas de bodega.

**Independent Test**: Puede probarse completando el formulario con datos válidos de remitente, destinatario, dirección y tipo de mercancía, ejecutando el pesaje incluido y confirmando el registro. El test es exitoso si el sistema genera un UUID único, emite la etiqueta digital del paquete en el sistema, el paquete queda en estado `Recibido en Sede` y los logs confirman que el evento `solicitar_ruta` fue enviado al `Módulo de Gestión de Rutas` con el payload completo.

**Acceptance Scenarios**:

1. **Scenario**: Registro exitoso con pesaje y solicitud de ruta automática
   - **Given** el empleado está autenticado, la sede está operativa y el cliente entrega el paquete con sus datos de contacto.
   - **When** el empleado completa todos los campos del formulario (ver FR-009), el sistema resuelve las coordenadas GPS vía Google Maps Geocoding API, se ejecuta obligatoriamente [Procesar Pesaje y Dimensiones (MOD1-UC-002)](./MOD1-UC-002-Procesar-Pesaje-Y-Dimensiones.md) con resultado exitoso, y el empleado confirma el registro.
   - **Then** el sistema genera un UUID (v4) único, registra `timestamp_ingreso` UTC e `id_sede`, actualiza el estado a `Recibido en Sede`, genera la etiqueta digital del paquete en el sistema e invoca automáticamente [Solicitar Ruta de Paquete (MOD1-UC-003)](./MOD1-UC-003-Solicitar-Ruta-De-Paquete.md).

2. **Scenario**: Registro con fallo del servicio de geolocalización — campo GPS pendiente
   - **Given** el empleado completa todos los campos del formulario pero la Google Maps Geocoding API no responde (timeout > 5 segundos) o retorna un error.
   - **When** el sistema detecta el fallo del servicio.
   - **Then** el sistema guarda la dirección en texto plano, marca `gps_estado = Pendiente GPS` en la entidad Paquete y notifica al empleado con un aviso visible. El paquete puede completar el pesaje y generarse la etiqueta digital, pero el evento `solicitar_ruta` queda bloqueado hasta que `gps_estado` cambie a `Resuelto`, ya que `latitud` y `longitud` son campos obligatorios del payload.

---

### Edge Cases

- What happens when la Google Maps Geocoding API falla (timeout > 5 s o error HTTP)? El sistema permite ingresar `latitud` y `longitud` manualmente. Al guardar el ingreso manual, el sistema valida que los valores estén en rango (latitud: −90 a 90; longitud: −180 a 180) y actualiza `gps_estado = Resuelto`, desbloqueando la emisión de `solicitar_ruta`.
- What happens when se intenta confirmar el registro sin haber completado el pesaje? El sistema bloquea la confirmación con un mensaje indicando que [Procesar Pesaje y Dimensiones (MOD1-UC-002)](./MOD1-UC-002-Procesar-Pesaje-Y-Dimensiones.md) es obligatorio y que sin él no es posible completar el registro.
- How does system handle un método de pago no soportado en la sede actual? El sistema valida `metodo_pago` contra el campo `metodos_pago_habilitados` de la entidad Sede y muestra los métodos disponibles, impidiendo guardar hasta seleccionar uno compatible.
- What happens when el empleado cancela el formulario después de generar un UUID preliminar? El registro queda en estado `Borrador`. El job de limpieza (FR-011) lo elimina automáticamente tras 30 minutos de inactividad. No se emite ningún evento externo desde un `Borrador`.
- What happens when el pesaje detecta una categoría de paquete que requiere autorización de supervisor (ej. Peligroso con dimensiones fuera de límite)? El flujo de admisión queda suspendido hasta que el supervisor valide y autorice el registro. Ver [MOD1-CATEGORIAS-Y-ESTADOS-PAQUETE.md](./MOD1-CATEGORIAS-Y-ESTADOS-PAQUETE.md) para las condiciones de cada categoría.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST invocar la **Google Maps Geocoding API** para convertir la dirección de destino a coordenadas `latitud` y `longitud`. El timeout de la llamada está configurado en 5 segundos. Si la API no responde o retorna error, el sistema marca `gps_estado = Pendiente GPS` y habilita ingreso manual de coordenadas.
- **FR-002**: System MUST generar un UUID (v4) único e irrepetible para cada paquete y validar su unicidad en la base de datos antes de persistir.
- **FR-003**: System MUST registrar automáticamente `timestamp_ingreso` (UTC) e `id_sede` sin intervención del empleado.
- **FR-004**: System MUST capturar `valor_declarado` y `metodo_pago` como campos obligatorios. El `valor_declarado` es el monto en moneda local que el remitente declara como valor comercial del contenido; este valor establece el límite de responsabilidad del seguro y es la base para el cálculo de la tarifa de envío (junto con peso, volumen, tipo de mercancía y distancia, según la tabla de tarifas definida en MOD1-UC-002). El `metodo_pago` acepta únicamente los valores `prepago` o `contra_entrega`, validados contra `metodos_pago_habilitados` de la Sede.
- **FR-005**: System MUST validar que todos los campos obligatorios estén completos antes de permitir la confirmación. Los campos obligatorios son: datos del Remitente (ver FR-009), datos del Destinatario (ver FR-009), `direccion_destino_texto`, `valor_declarado` y `metodo_pago`.
- **FR-006**: System MUST generar una etiqueta digital del paquete al confirmar el registro. La etiqueta existe únicamente dentro del sistema como registro digital vinculado al UUID; incluye el UUID representado en código QR y código de barras 1D, los datos del remitente, destinatario y sede. No puede exportarse, descargarse ni imprimirse en ningún formato (PDF, imagen u otro).
- **FR-007**: System MUST impedir la edición de `uuid`, `timestamp_ingreso` e `id_sede` una vez que el registro pase de `Borrador` a `Recibido en Sede`.
- **FR-008**: System MUST permitir el ingreso manual de `latitud` y `longitud` como contingencia, validando rangos geográficos válidos (latitud: −90 a 90; longitud: −180 a 180). Al guardar valores manuales válidos, actualizar `gps_estado = Resuelto`.
- **FR-009**: System MUST capturar los siguientes campos de Remitente y Destinatario como obligatorios:
  - **Remitente**: `documento` (tipo + número), `nombre_completo`, `telefono`.
  - **Destinatario**: `documento` (tipo + número), `nombre_completo`, `telefono`, `email`, `direccion_entrega` (texto completo de la dirección de destino).
- **FR-010**: System MUST invocar [Procesar Pesaje y Dimensiones (MOD1-UC-002)](./MOD1-UC-002-Procesar-Pesaje-Y-Dimensiones.md) como parte obligatoria e inseparable del flujo de registro. La confirmación del registro queda bloqueada si el pesaje no fue completado exitosamente.
- **FR-011**: System MUST ejecutar un **scheduled job** cada 15 minutos que elimine permanentemente todos los registros de paquetes en estado `Borrador` con `timestamp_creacion` mayor a 30 minutos, liberando el UUID reservado.
- **FR-012**: System MUST invocar automáticamente [Solicitar Ruta de Paquete (MOD1-UC-003)](./MOD1-UC-003-Solicitar-Ruta-De-Paquete.md) inmediatamente después de que [Procesar Pesaje y Dimensiones (MOD1-UC-002)](./MOD1-UC-002-Procesar-Pesaje-Y-Dimensiones.md) complete exitosamente y el registro sea confirmado, siempre que `gps_estado = Resuelto`. Si `gps_estado = Pendiente GPS`, la invocación queda suspendida hasta que las coordenadas sean resueltas.

### Key Entities *(include if feature involves data)*

- **Paquete**: `uuid` (PK, v4), `estado` (enum), `timestamp_ingreso` (UTC), `id_sede` (FK), `gps_estado` (`Pendiente GPS` | `Resuelto`), `latitud` (decimal), `longitud` (decimal), `direccion_destino_texto` (texto), `valor_declarado` (decimal > 0), `metodo_pago` (`prepago` | `contra_entrega`), `prioridad` (`Estándar` | `Urgente`), `id_remitente` (FK), `id_destinatario` (FK).
- **Remitente**: `id` (PK), `tipo_documento` (CC | CE | PAS | NIT), `numero_documento`, `nombre_completo`, `telefono`.
- **Destinatario**: `id` (PK), `tipo_documento` (CC | CE | PAS | NIT), `numero_documento`, `nombre_completo`, `telefono`, `email`, `direccion_entrega`.
- **Atributos Físicos del Paquete** (subentidad de Paquete): `peso_kg` (decimal 2d), `largo_cm` (entero > 0), `ancho_cm` (entero > 0), `alto_cm` (entero > 0), `volumen_m3` (decimal 4d), `peso_volumetrico_kg` (decimal 2d), `tipo_mercancia` (`Estándar` | `Frágil` | `Peligroso`), `es_irregular` (boolean), `categoria_carga` (`Normal` | `Carga Especial`).
- **Sede**: `id` (PK), `codigo`, `nombre`, `direccion`, `ciudad`, `latitud`, `longitud`, `capacidad_peso_max_kg`, `capacidad_volumen_max_m3`, `tipo_sede` (`Principal` | `Auxiliar`), `estado` (`Activa` | `Inactiva`), `horario_operacion`, `metodos_pago_habilitados` (lista).
- **Historial de Estados**: `id`, `uuid_paquete` (FK), `estado_anterior`, `estado_nuevo`, `timestamp` (UTC), `id_usuario`.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los paquetes confirmados deben tener UUID único y los datos completos de remitente y destinatario persistidos.
- **SC-002**: El 95% de las direcciones deben resolverse vía Google Maps Geocoding API en tiempo real durante el registro (latencia < 5 s).
- **SC-003**: El 100% de los registros exitosos con `gps_estado = Resuelto` deben emitir el evento `solicitar_ruta` al `Módulo de Gestión de Rutas` de forma inmediata tras el pesaje exitoso.
- **SC-004**: El 100% de los `Borrador` con más de 30 minutos de antigüedad deben ser eliminados por el job de limpieza en cada ejecución.
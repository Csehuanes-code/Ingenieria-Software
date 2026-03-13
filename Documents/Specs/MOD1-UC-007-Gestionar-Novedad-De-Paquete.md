# Feature Specification: Gestionar Novedad de Paquete (MOD1-UC-007)

**Created**: 2026-02-28

## User Scenarios & Testing

### User Story 7 — Registro y procesamiento de novedades del ciclo de vida del paquete (P1)

Como Controlador de Novedades, necesito recibir y procesar los estados y novedades que provienen del Módulo 2 (Gestión de Rutas) cuando un paquete está en ruta (`En Tránsito`, `En Parada de Entrega`, `Entregado`) o presenta novedades operativas en campo (`Extraviado`, `Devolución`, `Dañado`), así como las novedades detectadas en bodega, registrando las anomalías con evidencia adjunta, para mantener actualizada la trazabilidad y permitir que el Módulo de Gestión de Finanzas consulte esta información mediante comunicación síncrona.

**Why this priority**: Es el soporte legal ante penalizaciones y cobros de pólizas, y el cierre contable del ciclo. Este caso de uso es el receptor principal de los estados que provienen del Módulo 2 mientras el paquete está en ruta. Sin el registro formal con evidencia, el Módulo de Gestión de Finanzas no puede ejecutar los ajustes financieros correspondientes.

**Importante sobre comunicación con otros módulos**:
- La comunicación con el `Módulo de Gestión de Finanzas` es **síncrona**. El módulo de gestión de paquetes expone un endpoint HTTP que el módulo de finanzas consume para consultar el estado del paquete ante una novedad: `GET /route/{idRoute}/package/{idPaquete}`.

**Independent Test**: Registrar una novedad de tipo `Dañado` con fotografía en un paquete activo, y verificar que el sistema notifique al remitente y destinatario, vincule la novedad al UUID en el historial y envíe automáticamente el payload al Módulo de Gestión de Finanzas con estado final.

**Acceptance Scenarios**:

1. **Recepción de estado "En Tránsito" desde Módulo 2**
   - **Given** el Módulo de Gestión de Rutas notifica que una ruta pasó a estado `En Tránsito`.
   - **When** el sistema recibe el evento del Módulo 2.
   - **Then** el sistema actualiza el estado del paquete a `En Tránsito`, registra la transición en el historial con fecha/hora e identificador del controlador de novedades, y envía notificación al destinatario informando que su paquete está en camino.

2. **Recepción de estado "En Parada de Entrega" desde Módulo 2**
   - **Given** el Módulo de Gestión de Rutas notifica que el transportador está en el sitio de entrega.
   - **When** el sistema recibe el evento del Módulo 2.
   - **Then** el sistema actualiza el estado del paquete a `En Parada de Entrega`, registra la transición en el historial con fecha/hora e identificador del controlador de novedades y envía notificación al destinatario informando que el transportador está llegando.

3. **Recepción de estado "Entregado" desde Módulo 2 con disponibilidad para consulta financiera**
   - **Given** el Módulo de Gestión de Rutas notifica que el paquete fue entregado exitosamente con firma y evidencia (POD).
   - **When** el Controlador de Novedades procesa el evento.
   - **Then** el sistema actualiza el estado a `Entregado`, vincula la evidencia (firma y foto POD), **expone la información del paquete** mediante el endpoint `GET /route/{idRoute}/package/{idPaquete}` para que el Módulo de Gestión de Finanzas pueda consultarla de forma síncrona.

4. **Registro de daño con evidencia desde campo o bodega**
   - **Given** un paquete con daños físicos visibles (reportado desde Módulo 2 en ruta o desde bodega por el Almacenista a través de [Actualizar Estado de Paquete por Novedad (MOD1-UC-006)](./MOD1-UC-006-Actualizar-Estado-De-Paquete-Por-Novedad.md)).
   - **When** el Controlador selecciona tipo `Dañado` y adjunta la evidencia fotográfica.
   - **Then** el sistema actualiza el estado del paquete a `Novedad en Bodega - Dañado`, vincula la evidencia, notifica a remitente y destinatario, y **expone la información del paquete** mediante el endpoint `GET /route/{idRoute}/package/{idPaquete}` para que el Módulo de Gestión de Finanzas pueda consultarla de forma síncrona.

5. **Declaración de extravío**
   - **Given** un paquete no encontrado físicamente (en bodega o reportado por el Módulo 2 en ruta).
   - **When** el Controlador registra la novedad como `Extraviado`.
   - **Then** el sistema actualiza el estado a `Novedad en Bodega - Extraviado`, registra la incidencia, notifica a remitente y destinatario, y **expone la información del paquete** mediante el endpoint `GET /route/{idRoute}/package/{idPaquete}` para que el Módulo de Gestión de Finanzas pueda consultarla de forma síncrona.

6. **Procesamiento de devolución**
   - **Given** un paquete retornado por el Módulo de Gestión de Rutas (dirección incorrecta, cliente ausente, zona de difícil acceso / orden público, rechazado por el cliente).
   - **When** el Controlador registra la novedad como `Devolución`.
   - **Then** el sistema actualiza el estado a `Novedad en Bodega - Devolución`, notifica a remitente y destinatario informando que el paquete será retornado a la sede, y **expone la información del paquete** mediante el endpoint `GET /route/{idRoute}/package/{idPaquete}` para que el Módulo de Gestión de Finanzas pueda consultarla de forma síncrona.

### Análisis post-devolución

Una vez recibido el paquete devuelto en bodega, se realiza una inspección física. El estado siguiente depende del resultado de dicha inspección. En todos los casos, el **destino final del paquete es la sede de origen**:

| Resultado del análisis | Estado resultante |
|---|---|
| Paquete en buen estado | `En Clasificación` (por defecto) |
| Daño detectado | `Novedad en Bodega - Dañado` |
| Requiere instrucción del remitente | `En Espera de Instrucción` |
| Contenido parcialmente extraviado | `Novedad en Bodega - Extraviado` |

### Edge Cases

- **¿Qué ocurre si el Módulo de Gestión de Finanzas no puede consultar el endpoint porque no está disponible?** El sistema mantiene la novedad registrada localmente y el estado del paquete actualizado. El endpoint permanece disponible para reintentos posteriores por parte del Módulo de Finanzas.
- **¿Cómo maneja el sistema la recepción de un evento duplicado proveniente del Módulo 2 para el mismo paquete?** El sistema valida el timestamp y UUID del evento. Si detecta que el evento ya fue procesado, lo descarta sin generar una segunda transición en el historial, previniendo inconsistencias en la trazabilidad.
- **¿Qué ocurre si la notificación al remitente o destinatario falla al registrar una novedad?** El sistema registra el fallo de notificación en el log de eventos y reintenta el envío. La novedad queda registrada en el historial con independencia del resultado de la notificación.

---

## Requirements

### Functional Requirements

- **FR-001**: Recibir y procesar automáticamente los eventos del Módulo de Gestión de Rutas cuando un paquete pasa a `En Tránsito`, `En Parada de Entrega`, `Entregado` o presenta novedades operativas en campo (`Extraviado`, `Devolución`, `Dañado`).
- **FR-002**: Notificar al remitente (vía teléfono) y al destinatario (vía teléfono y correo electrónico) al registrar cualquier novedad.
- **FR-003**: El estado por defecto de un paquete devuelto tras la inspección en bodega es `En Clasificación`.
- **FR-004**: Restringir el cambio del tipo de novedad.
- **FR-005**: Exponer un endpoint HTTP para comunicación **síncrona** con el Módulo de Gestión de Finanzas: `GET /route/{idRoute}/package/{idPaquete}`. Este endpoint debe devolver un payload JSON con:
  - `idRoute`: Identificador de la ruta
  - `idPaquete`: UUID del paquete
  - `estado`: Estado actual del paquete
- **FR-006**: El endpoint debe estar disponible para consultas del Módulo de Gestión de Finanzas en cualquier momento tras un cambio de estado.
- **FR-007**: Validar que el endpoint devuelva código HTTP 200 OK con el payload cuando el paquete existe, o HTTP 404 Not Found cuando no existe.
- **FR-008**: Detectar y prevenir el procesamiento de eventos duplicados del Módulo 2 mediante validación de timestamp y UUID.
- **FR-009**: Una devolución de ruta solo se puede realizar desde el Módulo de Gestión de Rutas. No se permiten devoluciones originadas desde bodega.

### Key Entities

- **Novedad**: tipo (Dañado / Extraviado / Devolución), origen (Bodega / Ruta), descripción, evidencias adjuntas, fecha/hora, identificador del Controlador responsable, motivo específico.
- **Historial de Estados**: registro inmutable de todas las transiciones del paquete, incluyendo re-ingresos por devolución, con fecha/hora, usuario/módulo responsable.
- **Estado Final**: determina la acción financiera en el Módulo de Gestión de Finanzas.
- **Endpoint de Consulta**: `GET /route/{idRoute}/package/{idPaquete}` - API HTTP expuesta para que el Módulo de Gestión de Finanzas consulte el estado del paquete de forma **síncrona**. La comunicación con el Módulo de Gestión de Finanzas es **síncrona**, a diferencia de la comunicación asíncrona con el Módulo de Gestión de Rutas.

---

## Success Criteria

- **SC-001**: El 100% de las novedades quedan vinculadas al UUID y visibles en el historial con origen claramente identificado (Bodega / Ruta).
- **SC-002**: El 100% de las novedades de tipo `Dañado` tienen evidencia adjunta antes de guardarse.
- **SC-003**: El endpoint de consulta responde con código HTTP 200 OK y payload completo en menos de 500ms para el 95% de las peticiones.
- **SC-004**: El 100% de los eventos del Módulo 2 son procesados y registrados en el historial dentro de los 10 segundos posteriores a su recepción.
- **SC-005**: El endpoint devuelve HTTP 404 Not Found para paquetes inexistentes y HTTP 500 Internal Server Error cuando falta el valor declarado.
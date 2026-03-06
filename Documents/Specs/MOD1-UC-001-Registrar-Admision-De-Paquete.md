# Feature Specification: Registrar Admisión de Paquete (MOD1-UC-001)

**Created**: 2026-02-28

## User Scenarios & Testing

### User Story 1 — Ingreso inicial del paquete (P1)

Como Empleado de Envío y Recepción, necesito registrar los datos del remitente, destinatario y paquete para generar su identidad en el sistema y disparar la solicitud de ruta al completar el pesaje.

**Why this priority**: Es el punto de entrada. Sin este registro el paquete no existe para ningún otro módulo. La solicitud de ruta se emite inmediatamente tras el pesaje exitoso, momento en que el paquete ya tiene todos los datos necesarios.

**Independent Test**: Completar el formulario, ejecutar el pesaje y confirmar. El test pasa si se genera un UUID único, la etiqueta digital queda registrada en el sistema, el paquete queda en `Recibido en Sede` y los logs confirman el envío de `solicitar_ruta`.

**Acceptance Scenarios**:

1. **Registro exitoso**
   - **Given** el empleado está autenticado y el cliente entrega el paquete con sus datos.
   - **When** el empleado llena el formulario, el GPS resuelve coordenadas vía Google Maps Geocoding API, se ejecuta [MOD1-UC-002](./MOD1-UC-002-Procesar-Pesaje-Y-Dimensiones.md) exitosamente y se confirma el registro.
   - **Then** el sistema genera un UUID único, registra fecha/hora y sede automáticamente, actualiza el estado a `Recibido en Sede`, genera la etiqueta digital e invoca [MOD1-UC-003](./MOD1-UC-003-Solicitar-Ruta-De-Paquete.md).

2. **Fallo de geolocalización**
   - **Given** la Google Maps Geocoding API no responde (timeout > 5 s).
   - **When** el sistema detecta el fallo.
   - **Then** guarda la dirección en texto plano, marca el GPS como pendiente y notifica al empleado. El registro puede completarse, pero `solicitar_ruta` queda bloqueado hasta resolver las coordenadas.

### Edge Cases

- **GPS falla**: el empleado puede ingresar coordenadas manualmente (lat −90/90, lon −180/180). Al guardarlas el bloqueo de `solicitar_ruta` se levanta.
- **Sin pesaje**: el sistema bloquea la confirmación.
- **Método de pago no soportado en la sede**: el sistema muestra los métodos válidos e impide guardar.
- **Formulario cancelado**: el registro queda en `Borrador` y se elimina a los 30 minutos.
- **Mercancía peligrosa fuera de límites**: el flujo queda suspendido hasta autorización del Supervisor de Admisión.

---

## Requirements

### Functional Requirements

- **FR-001**: Invocar Google Maps Geocoding API (timeout 5 s) para convertir la dirección en coordenadas. Ante fallo, habilitar ingreso manual y marcar GPS como pendiente.
- **FR-002**: Generar un UUID v4 único por paquete, verificando que no exista antes de persistirlo.
- **FR-003**: Registrar fecha/hora de ingreso (UTC) y sede automáticamente, sin intervención del empleado.
- **FR-004**: Capturar como obligatorios el **valor declarado** (monto que el remitente asigna al contenido; define el límite del seguro y es base de la tarifa) y el **método de pago** (prepago o contra entrega, validado según los métodos habilitados en la sede).
- **FR-005**: Validar que todos los campos obligatorios estén completos antes de permitir confirmar.
- **FR-006**: Generar una etiqueta digital del paquete vinculada al UUID, visible solo dentro del sistema (incluye código QR y código de barras). No puede exportarse, descargarse ni imprimirse.
- **FR-007**: Bloquear la edición de UUID, fecha de ingreso y sede una vez confirmado el registro.
- **FR-008**: Permitir ingreso manual de coordenadas como contingencia ante fallo de GPS, validando rangos geográficos válidos.
- **FR-009**: Capturar como obligatorios:
  - **Remitente**: tipo y número de documento, nombre completo, teléfono.
  - **Destinatario**: tipo y número de documento, nombre completo, teléfono, correo electrónico, dirección de entrega.
- **FR-010**: Invocar [MOD1-UC-002](./MOD1-UC-002-Procesar-Pesaje-Y-Dimensiones.md) como parte obligatoria. Sin pesaje exitoso no se puede confirmar el registro.
- **FR-011**: Ejecutar un job cada 15 minutos que elimine registros en `Borrador` con más de 30 minutos de inactividad.
- **FR-012**: Invocar [MOD1-UC-003](./MOD1-UC-003-Solicitar-Ruta-De-Paquete.md) automáticamente al confirmar el registro, siempre que el GPS esté resuelto.

### Key Entities

- **Paquete**: UUID, estado, fecha/hora de ingreso, sede, estado GPS, coordenadas, dirección de destino, valor declarado, método de pago.
- **Remitente**: tipo y número de documento, nombre completo, teléfono.
- **Destinatario**: tipo y número de documento, nombre completo, teléfono, correo electrónico, dirección de entrega.
- **Sede**: nombre, ciudad, tipo (Principal / Auxiliar), métodos de pago habilitados, capacidades máximas de peso y volumen.
- **Historial de Estados**: registro inmutable de cada transición con fecha/hora y usuario responsable.

---

## Success Criteria

- **SC-001**: El 100% de los paquetes confirmados tienen UUID único y datos completos de remitente y destinatario.
- **SC-002**: El 95% de las direcciones se resuelven vía Google Maps en tiempo real (< 5 s).
- **SC-003**: El 100% de los registros exitosos con GPS resuelto emiten `solicitar_ruta` de forma inmediata.
- **SC-004**: El 100% de los `Borrador` con más de 30 minutos se eliminan en cada ejecución del job.
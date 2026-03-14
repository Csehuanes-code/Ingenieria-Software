# Feature Specification: Registrar Admisión de Paquete (MOD1-UC-001)

**Created**: 2026-02-28

## User Scenarios & Testing

### User Story 1 — Ingreso inicial del paquete (P1)

Como Empleado de Envío y Recepción, necesito registrar los datos del remitente, destinatario y paquete para generar su identidad en el sistema y disparar la solicitud de ruta al completar el pesaje.

**Why this priority**: Es el punto de entrada. Sin este registro el paquete no existe para ningún otro módulo. La solicitud de ruta se emite inmediatamente tras el pesaje exitoso, momento en que el paquete ya tiene todos los datos necesarios.

**Independent Test**: Completar el formulario, ejecutar el pesaje y confirmar. El test pasa si se genera un UUID único, el paquete queda en `Recibido en Sede` y los logs confirman el envío de `solicitar_ruta`.

**Acceptance Scenarios**:

1. **Registro exitoso**
   - **Given** el empleado está autenticado y el cliente entrega el paquete con sus datos.
   - **When** el empleado llena el formulario, el GPS resuelve coordenadas vía Google Maps Geocoding API, se ejecuta [Procesar Pesaje y Dimensiones (MOD1-UC-002)](./MOD1-UC-002-Procesar-Pesaje-Y-Dimensiones.md) exitosamente y se confirma el registro.
   - **Then** el sistema genera un UUID único, registra fecha/hora y sede automáticamente, actualiza el estado a `Recibido en Sede` e invoca [Solicitar Ruta de Paquete (MOD1-UC-003)](./MOD1-UC-003-Solicitar-Ruta-De-Paquete.md).

2. **Dirección fuera de cobertura o mal ingresada**
   - **Given** el empleado ingresa una dirección que está fuera de la zona de cobertura del sistema o tiene errores de formato.
   - **When** el sistema valida la dirección y detecta que está fuera de cobertura o mal formateada.
   - **Then** el sistema genera una **alerta bloqueante** que impide continuar el flujo hasta que se corrija la dirección o se confirme que está dentro de la cobertura del servicio.

3. **Fallo de geolocalización**
   - **Given** la Google Maps Geocoding API no responde (timeout > 5 s).
   - **When** el sistema detecta el fallo.
   - **Then** guarda la dirección en texto plano, marca el GPS como pendiente y notifica al empleado. El registro puede completarse, pero `solicitar_ruta` queda bloqueado hasta resolver las coordenadas.

### Edge Cases

- **¿Qué ocurre si la API de Google Maps Geocoding falla o supera el timeout de 5 segundos?** El empleado puede ingresar coordenadas manualmente (lat −90/90, lon −180/180). Al guardarlas, el bloqueo de `solicitar_ruta` se levanta automáticamente.
- **¿Qué sucede si el empleado intenta confirmar el registro sin haber completado el pesaje?** El sistema bloquea la confirmación hasta que [Procesar Pesaje y Dimensiones (MOD1-UC-002)](./MOD1-UC-002-Procesar-Pesaje-Y-Dimensiones.md) se ejecute exitosamente.
- **¿Cómo maneja el sistema un método de pago que no está habilitado en la sede actual?** El sistema muestra al empleado únicamente los métodos válidos para esa sede e impide guardar el registro con un método no soportado.que se ingresen valores dentro del rango permitido.

---

## Requirements

### Functional Requirements

- **FR-001**: **[CRÍTICO - Validación de Cobertura]** Validar que la dirección esté dentro de la cobertura del servicio antes de cualquier procesamiento. Si está fuera de cobertura o mal ingresada, generar alerta bloqueante que impida continuar el flujo hasta que se corrija.
- **FR-002**: Invocar Google Maps Geocoding API (timeout 5 s) para convertir la dirección en coordenadas. Ante fallo, habilitar ingreso manual y marcar GPS como pendiente.
- **FR-003**: Generar un UUID v4 único por paquete, verificando que no exista antes de persistirlo.
- **FR-004**: Registrar fecha/hora de ingreso (UTC) y sede automáticamente, sin intervención del empleado.
- **FR-005**: Capturar como obligatorios el **valor declarado** (monto que el remitente asigna al contenido; define el límite del seguro y es base de la tarifa) y el **método de pago** (prepago o contra entrega, validado según los métodos habilitados en la sede).
- **FR-006**: Validar que todos los campos obligatorios estén completos antes de permitir confirmar.
- **FR-007**: Generar una etiqueta digital del paquete vinculada al UUID, visible solo dentro del sistema. No puede exportarse, descargarse ni imprimirse. **No se generan códigos QR ni de barras**.
- **FR-008**: Bloquear la edición de UUID, fecha de ingreso y sede una vez confirmado el registro.
- **FR-009**: Permitir ingreso manual de coordenadas como contingencia ante fallo de GPS, validando rangos geográficos válidos.
- **FR-010**: Capturar como obligatorios:
  - **Remitente**: tipo y número de documento, nombre completo, teléfono.
  - **Destinatario**: tipo y número de documento, nombre completo, teléfono, correo electrónico, dirección de entrega.
- **FR-011**: Invocar [Procesar Pesaje y Dimensiones (MOD1-UC-002)](./MOD1-UC-002-Procesar-Pesaje-Y-Dimensiones.md) como parte obligatoria. Sin pesaje exitoso no se puede confirmar el registro.
- **FR-012**: Invocar [Solicitar Ruta de Paquete (MOD1-UC-003)](./MOD1-UC-003-Solicitar-Ruta-De-Paquete.md) automáticamente al confirmar el registro, siempre que el GPS esté resuelto. La comunicación con el Módulo de Gestión de Rutas es **asíncrona** mediante payloads JSON y las solicitudes se generan **una sola vez**.
- **FR-013**: Una vez que el Módulo de Gestión de Rutas asigna la ruta, el Empleado de Envío y Recepción debe informar al cliente (remitente) que el paquete tiene una **fecha estimada máxima de entrega de 7 días hábiles**.

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
- **SC-003**: El 100% de los registros exitosos con GPS resuelto emiten [Solicitar Ruta de Paquete (MOD1-UC-003)](./MOD1-UC-003-Solicitar-Ruta-De-Paquete.md) de forma inmediata y asíncrona (una sola vez).
- **SC-004**: El 100% de las direcciones fuera de cobertura o mal ingresadas generan alerta bloqueante que impide continuar el flujo.
- **SC-005**: El 100% de los clientes (remitentes) son informados del SLA de entrega de 7 días hábiles cuando la ruta es asignada exitosamente.
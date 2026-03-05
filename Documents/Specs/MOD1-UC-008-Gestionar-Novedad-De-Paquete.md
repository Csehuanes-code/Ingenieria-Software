# Feature Specification: Gestionar Novedad de Paquete (MOD1-UC-008)

**Created**: 2026-02-28

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Registro formal de incidencias con evidencia y notificación a partes (Priority: P1)

Como Controlador de Novedades, necesito registrar formalmente las anomalías de los paquetes (Daño, Extravío, Devolución) con evidencia adjunta cuando aplique, notificar al remitente y destinatario vía mensaje, y activar automáticamente la comunicación financiera a través de [Informar Estado de Paquete (MOD1-UC-009)](./MOD1-UC-009-Informar-Estado-De-Paquete.md), para mantener la trazabilidad del paquete y permitir el cierre contable del ciclo de vida.

**Why this priority**: Es la garantía de integridad de la mercancía y el soporte legal ante penalizaciones a transportadores o cobros de pólizas de seguro. Sin el registro formal, el `Módulo de Gestión de Finanzas` no puede ejecutar ajustes financieros, y el remitente y destinatario no reciben información actualizada sobre el estado de su envío.

**Independent Test**: Puede probarse seleccionando un paquete en estado `Listo para Despacho` o `En Tránsito`, registrando una novedad de tipo `Dañado` con fotografía adjunta válida (JPEG < 10 MB), y verificando que: (1) el sistema persiste la URL de evidencia, (2) el estado del paquete se actualiza a `Dañado`, (3) se envían notificaciones vía mensaje al remitente y destinatario, y (4) se invoca MOD1-UC-009 enviando al `Módulo de Gestión de Finanzas` UUID, `valor_declarado` y `url_evidencia`.

**Acceptance Scenarios**:

1. **Scenario**: Registro de avería física con evidencia obligatoria
   - **Given** un paquete identificado por UUID que presenta daños físicos visibles.
   - **When** el Controlador selecciona el tipo de novedad `Dañado`, adjunta al menos un archivo de evidencia válido (JPEG/PNG/MP4, ≤ 10 MB) y confirma el registro.
   - **Then** el sistema sube la evidencia al servicio de almacenamiento en la nube, persiste la URL pre-firmada, actualiza el estado del paquete a `Dañado`, envía notificaciones vía mensaje al remitente (al `telefono` registrado) y al destinatario (al `telefono` y `email` registrados) informando del daño, e invoca obligatoriamente [Informar Estado de Paquete (MOD1-UC-009)](./MOD1-UC-009-Informar-Estado-De-Paquete.md).

2. **Scenario**: Declaración de extravío
   - **Given** un paquete que no se encuentra físicamente tras el proceso de clasificación o carga.
   - **When** el Controlador selecciona el tipo de novedad `Extraviado` y confirma el registro.
   - **Then** el sistema registra la incidencia vinculada al UUID, actualiza el estado a `Extraviado`, envía notificaciones vía mensaje al remitente y al destinatario informando del extravío e invoca [Informar Estado de Paquete (MOD1-UC-009)](./MOD1-UC-009-Informar-Estado-De-Paquete.md) para iniciar la indemnización.

3. **Scenario**: Procesamiento de devolución — análisis de estado previo al re-ingreso
   - **Given** un paquete retornado a sede por el `Módulo de Gestión de Rutas` (dirección errónea, cliente no encontrado, rechazo de la entrega, etc.).
   - **When** el Controlador registra el re-ingreso con tipo de novedad `Devolución`.
   - **Then** el sistema actualiza el estado del paquete a `Devolución` y genera una tarea de análisis obligatoria: el Controlador debe inspeccionar físicamente el paquete y seleccionar el resultado del análisis (ver FR-004). El estado posterior al análisis depende del resultado. Adicionalmente, el sistema envía notificaciones vía mensaje al remitente y destinatario informando la devolución e invoca [Informar Estado de Paquete (MOD1-UC-009)](./MOD1-UC-009-Informar-Estado-De-Paquete.md) para el ajuste financiero de logística inversa.

---

### Edge Cases

- What happens when se intenta registrar una novedad de tipo `Dañado` sin adjuntar evidencia? El sistema bloquea el guardado. Al menos un archivo válido (JPEG, PNG o MP4, ≤ 10 MB) es obligatorio para tipo `Dañado`. No se permite continuar hasta adjuntar evidencia.
- What happens when se adjunta un archivo con formato no permitido o que supera el tamaño máximo? El sistema rechaza el archivo específico y muestra el mensaje de error por archivo, manteniendo el formulario abierto para que el usuario seleccione un archivo válido.
- What happens when un paquete devuelto presenta daños además de la devolución? El sistema permite registrar simultáneamente los tipos `Devolución` y `Dañado`. Se requiere evidencia fotográfica obligatoria (por el tipo `Dañado`). El análisis post-devolución tendrá en cuenta ambas condiciones. Se activan ambos protocolos financieros ante M3.
- What happens when el `Módulo de Gestión de Finanzas` no responde? El sistema persiste la novedad localmente y encola la notificación para reintento automático cada 5 minutos, dejando el paquete en estado `Pendiente Sincronización Contable`.
- What happens when se intenta modificar el tipo de novedad de un paquete ya en estado `Extraviado`? El sistema permite agregar nuevas evidencias o ampliar la descripción técnica. El cambio de tipo de novedad requiere autorización explícita de un usuario con rol `Supervisor de Novedades` o `Administrador del Sistema`. Los roles `Controlador de Novedades` y `Almacenista` no tienen permiso para cambiar el tipo de novedad de un registro ya persistido.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST permitir la carga de archivos de evidencia multimedia con las siguientes restricciones: formatos permitidos `JPEG`, `PNG`, `MP4`; tamaño máximo por archivo: 10 MB; máximo 5 archivos por novedad. Los archivos se almacenan en un servicio de almacenamiento en la nube (compatible con AWS S3 / GCP Cloud Storage). El sistema persiste únicamente la URL pre-firmada con tiempo de expiración de 1 año en el campo `url_evidencia` de la entidad Novedad.
- **FR-002**: System MUST bloquear el guardado de una novedad de tipo `Dañado` si no se adjuntó al menos un archivo de evidencia válido.
- **FR-003**: System MUST invocar obligatoriamente [Informar Estado de Paquete (MOD1-UC-009)](./MOD1-UC-009-Informar-Estado-De-Paquete.md) al finalizar cualquier registro de novedad exitoso (Dañado, Extraviado o Devolución).
- **FR-004**: System MUST implementar el flujo de análisis post-devolución: al registrar una novedad de tipo `Devolución`, el sistema genera una tarea de análisis obligatoria para el Controlador de Novedades. Los resultados posibles del análisis y los estados resultantes son:

  | Resultado del análisis | Estado del paquete tras el análisis |
  |---|---|
  | Paquete en buen estado, re-despacho inmediato viable | `En Clasificación` (por defecto; vuelve al flujo de bodega) |
  | Paquete con daños detectados durante la devolución | `Dañado` (se registra novedad adicional de tipo `Dañado`) |
  | Paquete requiere revisión adicional o instrucción del remitente | `En espera de instrucción` (estado de pausa; se notifica al remitente) |
  | Re-despacho no viable (extravío parcial del contenido) | `Extraviado` |

  El estado por defecto, si no se detecta ningún problema adicional, es `En Clasificación`, devolviendo el paquete al flujo de bodega para su reclasificación y nuevo despacho.

- **FR-005**: System MUST enviar notificaciones vía mensaje al **remitente** (al `telefono` registrado en la entidad Remitente) y al **destinatario** (al `telefono` y `email` registrados en la entidad Destinatario) en los siguientes momentos: (a) al registrar exitosamente cualquier novedad, informando el tipo de novedad y el estado actual del paquete; (b) al completar el análisis post-devolución, informando el estado resultante y los próximos pasos.
- **FR-006**: System MUST restringir la modificación del tipo de novedad de un registro ya persistido a usuarios con rol `Supervisor de Novedades` o `Administrador del Sistema`. Los roles `Controlador de Novedades`, `Almacenista` y `Coordinador de Despacho` pueden agregar evidencias adicionales o ampliar la descripción técnica, pero no pueden cambiar el tipo de novedad.

### Key Entities *(include if feature involves data)*

- **Novedad**: `id` (PK), `uuid_paquete` (FK), `tipo` (`Dañado` | `Extraviado` | `Devolución`), `descripcion_tecnica`, `url_evidencia` (lista de hasta 5 URLs pre-firmadas), `resultado_analisis` (nullable; para devoluciones), `timestamp` (UTC), `id_controlador` (FK), `estado_sincronizacion` (`Pendiente` | `Sincronizado` | `Error`).
- **Roles autorizados para registrar novedades**: `Controlador de Novedades`, `Supervisor de Novedades`, `Administrador del Sistema`.
- **Roles autorizados para modificar tipo de novedad**: `Supervisor de Novedades`, `Administrador del Sistema`.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las novedades registradas deben quedar vinculadas al UUID con URL de evidencia almacenada en la nube (para tipo `Dañado`) y visibles en el Historial de Estados.
- **SC-002**: El 100% de las novedades registradas deben disparar notificaciones vía mensaje al remitente y al destinatario.
- **SC-003**: El 100% de las devoluciones deben completar el análisis post-devolución antes de que el paquete avance a un nuevo estado operativo.
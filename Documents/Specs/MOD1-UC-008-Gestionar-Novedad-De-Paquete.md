# Feature Specification: Gestionar Novedad de Paquete (MOD1-UC-008)

**Created**: 2026-02-28

## User Scenarios & Testing

### User Story 1 — Registro formal de incidencias (P1)

Como Controlador de Novedades, necesito registrar las anomalías de los paquetes (Daño, Extravío, Devolución) con evidencia adjunta, para mantener actualizada la trazabilidad e invocar automáticamente al `Módulo de Gestión de Finanzas`.

**Why this priority**: Es el soporte legal ante penalizaciones y cobros de pólizas. Sin el registro formal con evidencia, el `Módulo de Gestión de Finanzas` no puede ejecutar los ajustes financieros correspondientes.

**Independent Test**: Registrar una novedad de tipo `Dañado` con fotografía en un paquete activo, y verificar que el sistema invoque [Informar Estado de Paquete(MOD1-UC-009)](./MOD1-UC-009-Informar-Estado-De-Paquete.md), notifique al remitente y destinatario, y que la novedad quede vinculada al UUID en el historial.

**Acceptance Scenarios**:

1. **Registro de daño con evidencia**
   - **Given** un paquete con daños físicos visibles.
   - **When** el Controlador selecciona tipo `Dañado` y adjunta la evidencia fotográfica.
   - **Then** el sistema actualiza el estado del paquete, vincula la evidencia, notifica a remitente y destinatario, e invoca [Informar Estado de Paquete(MOD1-UC-009)](./MOD1-UC-009-Informar-Estado-De-Paquete.md).

2. **Declaración de extravío**
   - **Given** un paquete no encontrado físicamente.
   - **When** el Controlador registra la novedad como `Extraviado`.
   - **Then** el sistema registra la incidencia, notifica a remitente y destinatario, e invoca [Informar Estado de Paquete(MOD1-UC-009)](./MOD1-UC-009-Informar-Estado-De-Paquete.md) para iniciar la indemnización.

3. **Procesamiento de devolución**
   - **Given** un paquete retornado por el `Módulo de Gestión de Rutas` (dirección errónea, destinatario no encontrado u otra causa).
   - **When** el Controlador registra la novedad como `Devolución`.
   - **Then** el sistema notifica a remitente y destinatario, invoca [Informar Estado de Paquete(MOD1-UC-009)](./MOD1-UC-009-Informar-Estado-De-Paquete.md) para el ajuste financiero, e inicia el análisis post-devolución para definir el nuevo estado del paquete.

### Análisis post-devolución

El estado siguiente depende del resultado de la inspección física:

| Resultado del análisis | Estado resultante |
|---|---|
| Paquete en buen estado | `En Clasificación` (por defecto) |
| Daño detectado | `Dañado` |
| Requiere instrucción del remitente | `En Espera de Instrucción` |
| Contenido parcialmente extraviado | `Extraviado` |

### Edge Cases

- **Daño sin evidencia**: el sistema bloquea el guardado hasta adjuntar al menos un archivo multimedia válido.
- **Devolución con daño simultáneo**: el sistema permite registrar ambas condiciones. Se activan logística inversa y gestión de seguro.
- **Novedad sobre paquete ya `Extraviado`**: se puede agregar información adicional (evidencias, descripción), pero cambiar el tipo de novedad requiere autorización del Supervisor de Novedades o Administrador del Sistema.
- **M3 no responde**: el sistema registra la novedad localmente, encola la notificación para reintento cada 5 minutos y deja el paquete en `Pendiente Sincronización Contable`.

---

## Requirements

### Functional Requirements

- **FR-001**: Permitir adjuntar archivos multimedia (fotos/videos) como evidencia obligatoria para novedades de tipo `Dañado`.
- **FR-002**: Invocar [Informar Estado de Paquete(MOD1-UC-009)](./MOD1-UC-009-Informar-Estado-De-Paquete.md) al finalizar cualquier registro de novedad, independientemente del tipo.
- **FR-003**: Notificar al remitente (vía teléfono) y al destinatario (vía teléfono y correo) al registrar la novedad y al completar el análisis post-devolución.
- **FR-004**: Aplicar el análisis post-devolución para definir el estado del paquete según la tabla anterior. El estado por defecto es `En Clasificación`.
- **FR-005**: Restringir el cambio del tipo de novedad a Supervisor de Novedades o Administrador del Sistema.

### Key Entities

- **Novedad**: tipo (Dañado / Extraviado / Devolución), descripción, evidencias adjuntas, fecha/hora, identificador del Controlador responsable.
- **Historial de Estados**: registro de todas las transiciones del paquete, incluyendo re-ingresos por devolución.

---

## Success Criteria

- **SC-001**: El 100% de las novedades quedan vinculadas al UUID y visibles en el historial.
- **SC-002**: El 100% de las novedades de tipo `Dañado` tienen evidencia adjunta antes de guardarse.
- **SC-003**: El 100% de las devoluciones pasan por análisis post-devolución antes de definir su nuevo estado.
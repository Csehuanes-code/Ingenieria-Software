# Feature Specification: Gestionar Novedad de Paquete (MOD1-UC-007)

**Created**: 2026-02-28

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Registro formal de incidencias con evidencia obligatoria (Priority: P1)

Como Controlador de Novedades, necesito registrar formalmente las anomalías de los paquetes (Daño, Extravío, Devolución) con evidencia fotográfica adjunta, para mantener la hoja de vida del paquete actualizada y activar automáticamente la comunicación con el Módulo 3 a través del caso de uso `Informar Estado de Paquete` (`<<include>>`).

**Why this priority**: Es la garantía de integridad de la mercancía y el soporte legal de la empresa ante penalizaciones a transportadores o cobros de pólizas de seguro. Sin el registro formal con evidencia, el Módulo 3 no puede ejecutar descuentos, cobros ni ajustes financieros sobre el paquete afectado.

**Independent Test**: Puede probarse seleccionando un paquete en estado `Listo para Despacho` o `En Tránsito`, registrando una novedad de tipo `Dañado` con fotografía adjunta, y verificando que el sistema invoque `Informar Estado de Paquete` enviando al Módulo 3 el UUID, el valor declarado y la URL de la evidencia.

**Acceptance Scenarios**:

1. **Scenario**: Registro de avería física con evidencia obligatoria
   - **Given** un paquete identificado por su UUID que presenta daños físicos visibles.
   - **When** el Controlador selecciona el tipo de novedad `Dañado` y adjunta la evidencia fotográfica requerida.
   - **Then** el sistema actualiza el estado interno del paquete, vincula la evidencia al UUID y ejecuta obligatoriamente el caso de uso `Informar Estado de Paquete` (`<<include>>`), enviando al Módulo 3 el UUID, el valor declarado y la URL de la evidencia para iniciar el proceso de penalización o cobro de póliza.

2. **Scenario**: Declaración de extravío en auditoría de bodega
   - **Given** un paquete que no se encuentra físicamente tras el proceso de clasificación o carga.
   - **When** el Controlador actualiza el estado del paquete a `Extraviado`.
   - **Then** el sistema dispara una alerta interna, registra la incidencia vinculada al UUID e invoca `Informar Estado de Paquete` (`<<include>>`) para iniciar el proceso de indemnización basado en el valor declarado ante el Módulo 3.

3. **Scenario**: Procesamiento de devolución por logística inversa
   - **Given** un paquete retornado por el Módulo 2 debido a causas externas como dirección errónea o cliente no encontrado.
   - **When** el Controlador registra el re-ingreso a bodega con el tipo de novedad `Devolución`.
   - **Then** el sistema cambia el estado del paquete a `Listo para Despacho` para habilitar su re-proceso en el flujo logístico, y ejecuta `Informar Estado de Paquete` (`<<include>>`) para que el Módulo 3 gestione el ajuste financiero correspondiente a la devolución.

---

### Edge Cases

- What happens when se intenta registrar una novedad de tipo `Dañado` sin adjuntar evidencia fotográfica? El sistema bloquea el guardado y muestra una alerta indicando que la evidencia es obligatoria para el tipo de novedad `Dañado`. No se permite continuar hasta adjuntar al menos un archivo multimedia válido.
- What happens when un paquete devuelto presenta daños además de la devolución? El sistema permite registrar simultáneamente las condiciones `Devolución` y `Dañado`. Se activan ambos protocolos: logística inversa para el re-despacho y gestión de seguro ante el Módulo 3.
- What happens when el Módulo 3 no responde al ser invocado por `Informar Estado de Paquete`? El sistema registra la novedad localmente y encola la notificación al Módulo 3 para reintento automático cada 5 minutos, dejando el paquete en estado `Pendiente Sincronización Contable` hasta recibir confirmación.
- What happens when se intenta registrar una novedad sobre un paquete que ya tiene el estado `Extraviado`? El sistema permite agregar información adicional (nuevas evidencias o descripción técnica ampliada) pero no permite cambiar el tipo de novedad sin autorización del supervisor.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST permitir la carga de archivos multimedia (fotos, videos) vinculados al UUID del paquete como evidencia obligatoria para novedades del tipo `Dañado`.
- **FR-002**: System MUST invocar obligatoriamente el caso de uso `Informar Estado de Paquete` (`<<include>>`) al finalizar cualquier registro de novedad, independientemente del tipo (Dañado, Extraviado o Devolución).
- **FR-003**: System MUST permitir la reclasificación de zona para paquetes en estado `Devolución`, habilitando su re-ingreso al flujo de despacho como `Listo para Despacho`.

### Key Entities *(include if feature involves data)*

- **Novedad**: Entidad que registra el tipo de incidencia (Dañado, Extraviado, Devolución), descripción técnica, URL de evidencias multimedia, timestamp y ID del Controlador responsable.
- **Log de Estado**: Historial de movimientos del UUID que permite visualizar todas las transiciones del ciclo de vida del paquete, incluyendo re-ingresos por devolución.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las novedades reportadas deben quedar vinculadas al UUID y ser visibles en el historial de trazabilidad del paquete.
- **SC-002**: Las devoluciones procesadas deben reflejarse en el inventario de `Listo para Despacho` en menos de 5 minutos tras su registro.
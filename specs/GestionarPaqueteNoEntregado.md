# Feature Specification: Gestionar Paquete No Entregado (MOD1-UC-007)

**Created**: 2026-02-26

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Resolución y direccionamiento de novedades (Priority: P3-Media)

Como Controlador de Novedades, necesito registrar el motivo por el cual un paquete no fue entregado para definir su siguiente paso operativo: reprogramación o devolución.

**Why this priority**: Evita que los paquetes queden en un "limbo logístico", restableciendo y manteniendo la cadena de custodia al definir una acción clara sobre el retorno.

**Independent Test**: Puede probarse simulando el evento de recepción de novedad desde el Módulo 2, permitiendo al usuario seleccionar "Reprogramar" (verificando incremento de contador) o "Devolver" (verificando generación de etiqueta inversa).

**Acceptance Scenarios**:

1. **Scenario**: Reprogramación de entrega por cliente ausente
   - **Given**: El paquete ingresa al estado 'Novedad en Bodega', posee historial de intentos y el Controlador está autenticado.
   - **When**: El Controlador selecciona la causal, elige la acción 'Reprogramar' validando que no se ha superado el límite de intentos.
   - **Then**: El sistema actualiza el estado a 'En Bodega - Reprogramado', incrementa el contador de intentos y habilita el flujo para una nueva asignación de ruta.

### Edge Cases

- What happens when: el paquete ha superado el límite máximo configurado de intentos (3)? El sistema bloquea la opción 'Reprogramar', fuerza la transición al estado 'Devolución a Origen' y genera etiquetas automáticamente.
- What happens when: el paquete sufrió daños físicos evidentes durante el intento de entrega? El sistema actualiza a 'Novedad en Bodega - Dañado' y bloquea la reprogramación/devolución hasta que un supervisor determine una acción legal o de seguro.
- How does system handle: si el cliente rechaza el pago contra entrega en puerta? Queda registrado el motivo 'rechazado_por_cliente', se notifica al Módulo 3 del cobro fallido, y el Controlador decide la acción según las políticas comerciales.
- What happens when: el Módulo 2 reporta un paquete como extraviado en ruta? Se actualiza el estado a 'Novedad en Bodega - Extraviado' y se activa automáticamente el protocolo de seguros, bloqueando el paquete para otras acciones.
- What happens when: se intenta reprogramar un paquete hacia una zona de destino que fue eliminada de la configuración? El sistema solicita y exige una reasignación manual de zona antes de permitir y habilitar la reprogramación.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST: proveer un catálogo estandarizado e inmutable de causales de no entrega en el sistema.
- **FR-002**: System MUST: mantener un contador de intentos de entrega por UUID, incrementándolo en cada evento de reprogramación.
- **FR-003**: System MUST: bloquear la opción de reprogramación cuando el contador alcance o supere el umbral predeterminado (default: 3 intentos).
- **FR-004**: System MUST: habilitar la extensión hacia el flujo de asignación de ruta (MOD1-UC-005) al registrar un paquete como reprogramado.
- **FR-005**: System MUST: notificar automáticamente al Módulo 3 si una no entrega afecta un pago contra entrega pendiente.
- **FR-006**: System MUST: generar las etiquetas correspondientes de logística inversa al cambiar el estado a 'Devolución a Origen'.
- **FR-007**: System MUST: registrar cada novedad almacenando UUID, motivo, subtipo, timestamp, ID del controlador y origen de la novedad.
- **NFR-001**: System MUST: asegurar que el 100% de los paquetes no entregados tengan causal y acción definida en menos de 12 horas desde el reporte.
- **NFR-002**: System MUST: garantizar que el historial de novedades sea estrictamente inmutable; solo puede ser complementado, nunca eliminado.

### Key Entities *(include if feature involves data)*

- **Novedad**: Entidad de registro que contiene motivo, subtipo, timestamp, origen y url de foto de evidencia.
- **Estado Logístico**: Atributo interno del paquete que refleja si la unidad está bloqueada, en reprogramación o en devolución.
- **Contador de Intentos**: Atributo que registra numéricamente cuántos intentos de entrega se han realizado para aplicar las reglas de negocio.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los paquetes con novedad deben tener una acción y paso definido en un máximo de 12 horas.
- **SC-002**: El sistema debe bloquear rigurosamente el 100% de los intentos de reprogramación que excedan el límite de umbral.
- **SC-003**: El 100% de las novedades afectando cobros contra entrega deben generar una notificación al Módulo 3 en un lapso menor a 5 minutos.
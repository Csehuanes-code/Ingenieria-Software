# Feature Specification: Clasificar Paquete por Zona de Destino (MOD1-UC-006)

**Created**: 2026-02-28

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Agrupación lógica por zona geográfica de destino (Priority: P2)

Como Almacenista, necesito asignar a cada paquete su zona de destino —una agrupación lógica geográfica basada en la proximidad del destinatario— y actualizar su estado a `Clasificado`, para que el Coordinador de Despacho pueda identificar físicamente los grupos de paquetes por zona y para que el `Módulo de Gestión de Rutas` maximice la eficiencia en la consolidación de carga de vehículos.

**Why this priority**: Permite reducir tiempos de carga de vehículos y mejorar la eficiencia del algoritmo de consolidación del `Módulo de Gestión de Rutas`. Este caso de uso es invocado por [Preparar Paquete para Almacenaje (MOD1-UC-004)](./MOD1-UC-004-Preparar-Paquete-Para-Almacenaje.md). Al completarse, el paquete alcanza el estado `Clasificado`, que es el prerrequisito para que [Actualizar Estado de Disponibilidad (MOD1-UC-005)](./MOD1-UC-005-Actualizar-Estado-De-Disponibilidad.md) lo transite a `Listo para Despacho`.

> **Distinción de dominios**: La **Zona de Almacenamiento** es el espacio físico en bodega donde se ubica el paquete (gestionada en MOD1-UC-004). La **Zona de Destino** es una clasificación lógica y geográfica que agrupa paquetes por proximidad del destinatario para optimizar la consolidación de carga de vehículos (gestionada en este caso de uso). Son entidades independientes; un paquete tiene ambas asignadas al completar el flujo de almacenaje.

**Independent Test**: Puede probarse con un paquete en estado `En Clasificación`, validando que el sistema calcule correctamente la zona de destino usando `latitud`, `longitud` y `radio_zona_km` (default 5 km), genere la etiqueta digital de zona en el sistema y que el estado del paquete cambie a `Clasificado` al confirmar.

**Acceptance Scenarios**:

1. **Scenario**: Asignación exitosa de zona de destino — estado Clasificado
   - **Given** el paquete está en estado `En Clasificación`, tiene `gps_estado = Resuelto` y las zonas de destino están configuradas en el sistema.
   - **When** el sistema calcula la zona de destino usando `latitud`, `longitud` y `radio_zona_km`; el almacenista confirma la zona sugerida.
   - **Then** el sistema asigna la zona de destino al paquete, genera la etiqueta digital de zona dentro del sistema, actualiza el estado del paquete a `Clasificado` y registra la transición en el Historial de Estados.

2. **Scenario**: Destino fuera de zonas configuradas — escalamiento supervisado
   - **Given** las coordenadas GPS del paquete no corresponden a ninguna zona de destino configurada en el sistema.
   - **When** el sistema ejecuta el cálculo de zona.
   - **Then** el sistema muestra el aviso `Destino fuera de zonas configuradas`, bloquea la asignación automática y notifica al Supervisor de Bodega para que asigne manualmente la zona más apropiada. La excepción queda documentada en el Historial de Estados del paquete.

---

### Edge Cases

- What happens when las coordenadas GPS no corresponden a ninguna zona configurada? El Supervisor de Bodega asigna la zona manualmente y documenta la excepción. El "Supervisor de Bodega" es el usuario con rol superior al Almacenista dentro del área de operaciones de bodega; en el sistema, corresponde al rol `Supervisor de Bodega` o `Administrador del Sistema`.
- What happens when la zona de destino alcanza su capacidad lógica máxima (cantidad máxima de paquetes asignables)? El sistema emite una alerta en tiempo real. Los paquetes siguientes para esa zona son redirigidos a la zona de desborde lógico configurada o se pausa la clasificación hasta que el Supervisor de Bodega tome acción. Esta condición es independiente de la saturación de la zona de almacenamiento física.
- How does system handle el cambio de dirección de destino después de que el paquete ya fue clasificado? El sistema reclasifica automáticamente la zona de destino si las nuevas coordenadas corresponden a una zona diferente, actualiza el estado a `En Clasificación` y notifica al almacenista para reubicar físicamente el paquete si la nueva zona implica un andén de carga diferente. Si la nueva dirección no tiene zona configurada, notifica al Supervisor de Bodega para asignación manual.
- What happens when un paquete `Frágil` o `Peligroso` se clasifica en una zona de destino sin vehículos aptos configurados? El sistema emite una alerta y solicita confirmación del Supervisor de Bodega antes de completar la asignación, registrando la excepción.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST calcular la zona de destino del paquete usando `latitud`, `longitud` y el `radio_zona_km` configurado (valor por defecto: 5 km, configurable por el Administrador del Sistema).
- **FR-002**: System MUST generar la etiqueta de zona como registro digital dentro del sistema al confirmar la clasificación. La etiqueta existe únicamente en el sistema; no puede exportarse, descargarse ni imprimirse en ningún formato (PDF, imagen u otro). Debe incluir: código de zona, nombre de zona, UUID del paquete y código QR del UUID.
- **FR-003**: System MUST emitir una alerta si un paquete `Frágil` o `Peligroso` intenta clasificarse en una zona de destino sin vehículos aptos, solicitando confirmación supervisada antes de confirmar.
- **FR-004**: System MUST validar que la zona de destino no haya superado su `capacidad_max_paquetes` antes de confirmar la asignación.
- **FR-005**: System MUST actualizar el estado del paquete a `Clasificado` al confirmar exitosamente la asignación, registrando la transición en el Historial de Estados con `timestamp` UTC e `id_usuario`.

### Key Entities *(include if feature involves data)*

- **Paquete**: Al completar este caso de uso, el paquete tiene estado `Clasificado` y los siguientes **atributos logísticos** completos:

  | Atributo logístico | Descripción |
  |---|---|
  | `uuid` | Identificador único del envío |
  | `id_zona_almacenamiento` | Zona física en bodega donde está ubicado |
  | `id_zona_destino` | Zona lógica geográfica para consolidación de carga |
  | `id_ruta` | Ruta asignada por M2 (puede estar pendiente si M2 aún no respondió) |
  | `id_transportador` | Transportador asignado por M2 (puede estar pendiente) |
  | `fecha_estimada_entrega` | Fecha estimada provista por M2 |
  | `latitud` / `longitud` | Coordenadas GPS del destinatario |
  | `direccion_destino_texto` | Dirección textual del destinatario |
  | `metodo_pago` | `prepago` o `contra_entrega` |
  | `valor_declarado` | Monto declarado por el remitente |
  | `precio_envio_calculado` | Tarifa calculada en MOD1-UC-002 |
  | `prioridad` | `Estándar` o `Urgente` |

- **Zona de Destino**: `id` (PK), `nombre`, `latitud_centro`, `longitud_centro`, `radio_zona_km`, `capacidad_max_paquetes`, `paquetes_actuales`, `estado` (`Disponible` | `Saturado`).
- **Etiqueta de Zona**: Registro digital dentro del sistema que contiene código de zona, nombre de zona, UUID del paquete y código QR. No exportable ni imprimible.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los paquetes que completan este caso de uso deben tener zona de destino asignada y estado `Clasificado`.
- **SC-002**: El 0% de los paquetes deben poder avanzar a `Listo para Despacho` sin haber alcanzado el estado `Clasificado` previamente.
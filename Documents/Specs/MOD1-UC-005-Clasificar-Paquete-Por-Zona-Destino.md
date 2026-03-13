# Feature Specification: Clasificar Paquete por Zona de Destino (MOD1-UC-005)

**Created**: 2026-02-28

## User Scenarios & Testing

### User Story 5 — Agrupación geográfica de paquetes en bodega (P2)

Como Almacenista, necesito asignar a cada paquete su zona de destino —una agrupación lógica basada en la proximidad geográfica del destinatario— para que el `Módulo de Gestión de Rutas` pueda optimizar la consolidación de carga por vehículo.

**Why this priority**: Reduce los tiempos de carga y mejora la eficiencia del algoritmo de rutas. Es invocado por [Preparar Paquete para Almacenaje (MOD1-UC-004)](./MOD1-UC-004-Preparar-Paquete-Para-Almacenaje.md) una vez que el paquete está en bodega.

**Independent Test**: Con un paquete en `En Clasificación`, verificar que el sistema identifique la ciudad de destino, asigne la zona de bodega correspondiente (ej. Norte, Occidente, Sur) y actualice automáticamente el estado a `Listo para Despacho`.

**Acceptance Scenarios**:

1. **Asignación exitosa de zona de destino**
   - **Given** el paquete está en `En Clasificación`.
   - **When** el sistema calcula la zona de destino y el almacenista confirma.
   - **Then** el sistema asigna la zona, actualiza la clasificación lógica del paquete dentro del sistema y actualiza el estado del paquete a `Listo para Despacho`.

### Edge Cases

- **¿Qué ocurre si la zona de destino calculada ya ha alcanzado su capacidad máxima?** Los paquetes siguientes se redirigen automáticamente a la zona de desborde configurada, garantizando que el flujo no se interrumpa.
- **¿Qué sucede si el sistema intenta clasificar un paquete de tipo `Frágil` o `Peligroso` en una zona de destino que no tiene la categoría adecuada?** El sistema bloquea la clasificación y exige que el almacenista seleccione manualmente una zona de destino con la categoría de manejo adecuada antes de poder continuar.

---

## Requirements

### Functional Requirements

- **FR-001**: Agrupar paquetes por proximidad geográfica.
- **FR-002**: Registrar la clasificación lógica de zona de destino dentro del sistema al confirmar. Incluye código de zona, nombre de zona y UUID del paquete.
- **FR-003**: Bloquear la clasificación de paquetes `Frágil` o `Peligroso` en zonas no aptas para su categoría.
- **FR-004**: Validar que la zona de destino no haya superado su capacidad máxima antes de confirmar.

### Key Entities

- **Zona de Destino**: nombre, límites geográficos, capacidad máxima y estado de ocupación. Es una clasificación lógica independiente de la Zona de Almacenamiento física.
- **Clasificación de Zona**: registro digital en el sistema con código de zona, nombre y UUID del paquete.

---

## Success Criteria

- **SC-001**: El 100% de los paquetes clasificados tienen zona de destino asignada antes de avanzar a `Listo para Despacho`.
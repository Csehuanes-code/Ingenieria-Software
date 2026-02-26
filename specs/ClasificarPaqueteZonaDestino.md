# Feature Specification: Clasificar en zona de almacenamiento por destino

**Created**: 2026-02-23

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Segmentación Geográfica de Bodega (Priority: P2)

Como Almacenista, quiero agrupar los paquetes físicamente según su zona de destino (Norte, Sur, Oriente, Occidente o Ciudad destino) para asegurar que el Módulo 2 pueda realizar la consolidación de carga de manera eficiente.

**Why this priority**: Es el paso que elimina la incertidumbre geográfica. Si un paquete no está clasificado por zona, el sistema de rutas no puede calcular la carga óptima del vehículo ni cumplir con el llenado por capacidad del 90%.

**Independent Test**: Se puede testear seleccionando un grupo de paquetes con diferentes coordenadas GPS y verificando que el sistema sugiera la "Zona de Almacenamiento" correcta, actualizando su ubicación en la base de datos.

**Acceptance Scenarios**:

1. **Scenario**: Asignación automática de zona por coordenadas.
* **Given** un paquete con coordenadas GPS (lat/long) registradas en el ingreso.
* **When** el almacenista escanea el paquete para clasificación.
* **Then** el sistema indica la zona de almacenamiento específica (ej. "Muelle 4 - Zona Norte") y actualiza el estado a "En Clasificación".

2. **Scenario**: Validación de capacidad de zona.
* **Given** una zona de almacenamiento que ha alcanzado su límite de volumen o peso máximo.
* **When** el almacenista intenta asignar un nuevo paquete a esa zona.
* **Then** el sistema emite una alerta y sugiere una zona de contingencia o desborde.

### Edge Cases

* ¿Qué sucede si las coordenadas GPS del paquete no coinciden con ninguna zona de despacho configurada en el sistema?
* ¿Cómo se gestiona un paquete que debe ser reclasificado porque el cliente cambió la dirección de entrega a último minuto?

## Requirements *(mandatory)*

### Functional Requirements

* **FR-001**: El sistema DEBE agrupar paquetes automáticamente basándose en la cercanía geográfica (coordenadas GPS) definida en el Módulo 1.
* **FR-002**: El sistema DEBE permitir la impresión de etiquetas de zona para identificar físicamente el grupo de paquetes.
* **FR-003**: El sistema DEBE emitir una alerta si un paquete "Frágil" o "Peligroso" se intenta almacenar en una zona no apta para su tipo de mercancía.

### Key Entities

* **Zona de Almacenamiento**: Clasificación lógica y física en bodega que sirve de entrada para el algoritmo de Consolidación de Carga.
* **Estado de Paquete**: Atributo que cambia de "Recibido en Sede" a "En Clasificación" durante este proceso.

## Success Criteria *(mandatory)*

### Measurable Outcomes

* **SC-001**: El 100% de los paquetes clasificados deben tener una zona asignada antes de pasar al estado "Listo para Despacho".
* **SC-002**: Reducción del tiempo de carga de vehículos en un 20% al tener la mercancía ya agrupada por la zona que operará el conductor.

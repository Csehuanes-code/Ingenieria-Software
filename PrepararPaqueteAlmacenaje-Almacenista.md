
### User Story 2 - Preparación y Clasificación en Almacén (Priority: P2)

Como **Almacenista**, quiero preparar el paquete para su almacenaje y clasificarlo por zona de destino para agilizar el proceso de despacho masivo.

**Why this priority**: Una bodega organizada reduce los tiempos de carga en el Módulo 2.

**Independent Test**: Consultar el inventario de bodega filtrado por zona de destino y verificar que los paquetes listados correspondan a esa área geográfica.

**Acceptance Scenarios**:

1. **Scenario**: Clasificación por destino.
* **Given** un paquete en estado "Recibido en Sede".
* **When** el almacenista asigna el paquete a una zona específica de la bodega basada en el destino final.
* **Then** el sistema actualiza el estado a "En Clasificación".

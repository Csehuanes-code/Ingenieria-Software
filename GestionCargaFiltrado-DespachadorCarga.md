### User Story 3 - Gestión de Carga y Filtrado (Priority: P2)

Como **Despachador de carga**, quiero filtrar los paquetes por zona para gestionar la carga de los vehículos de manera eficiente.

**Why this priority**: Permite la consolidación de carga necesaria para optimizar el uso de la flota.

**Independent Test**: Realizar un filtro por "Zona Norte" y confirmar que solo se muestran paquetes con destinos en dicha zona y estado "Listo para Despacho".

**Acceptance Scenarios**:

1. **Scenario**: Filtrado para asignación.
* **Given** una lista de 100 paquetes en bodega.
* **When** el despachador aplica el filtro por zona geográfica.
* **Then** el sistema muestra solo los paquetes que cumplen el criterio para ser cargados en la ruta actual.


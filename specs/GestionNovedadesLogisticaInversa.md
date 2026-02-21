### User Story 4 - Gestión de Novedades y Logística Inversa (Priority: P3)

Como **Controlador de calidad**, quiero gestionar los paquetes no entregados para determinar si deben ser reintentados, devueltos o reportados como siniestro.

**Why this priority**: Gestiona la satisfacción del cliente y la responsabilidad financiera sobre la mercancía.

**Independent Test**: Cambiar un paquete a estado "Dañado" y verificar que se dispare una notificación al módulo de finanzas.

**Acceptance Scenarios**:

1. **Scenario**: Registro de novedad en ruta.
* **Given** un paquete en estado "En Tránsito" que no pudo ser entregado.
* **When** el controlador registra el motivo (Ej: Dirección incorrecta o Daño).
* **Then** el sistema actualiza el estado a "Novedad en Bodega" e inicia el flujo de devolución si aplica.

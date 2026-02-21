### User Story 1 - Registro y Procesamiento Físico (Priority: P1)

Como **Empleado de envío y recepción**, quiero registrar la admisión del paquete y procesar sus dimensiones físicas para asegurar que la base de datos logística sea exacta desde el origen.

**Why this priority**: Es la base de toda la operación. Sin dimensiones y peso precisos, no se puede calcular el costo ni asignar un vehículo adecuado.

**Independent Test**: Registrar un paquete y verificar que el sistema calcule el volumen y genere el UUID único automáticamente.

**Acceptance Scenarios**:

1. **Scenario**: Registro de admisión exitoso.
* **Given** un paquete físico en la ventanilla de recepción.
* **When** el empleado ingresa el origen, destino y tipo de mercancía.
* **Then** el sistema asigna un UUID y marca el estado como "Recibido en Sede".


2. **Scenario**: Procesamiento de pesaje y dimensiones.
* **Given** un paquete registrado en el sistema.
* **When** se capturan los kg y las medidas (largo, ancho, alto).
* **Then** el sistema actualiza los atributos físicos y calcula el volumen total.


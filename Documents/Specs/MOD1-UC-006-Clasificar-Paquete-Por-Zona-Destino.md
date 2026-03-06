# Feature Specification: Clasificar Paquete por Zona de Destino (MOD1-UC-006)

**Created**: 2026-02-28

## User Scenarios & Testing

### User Story 1 — Agrupación geográfica de paquetes en bodega (P2)

Como Almacenista, necesito asignar a cada paquete su zona de destino —una agrupación lógica basada en la proximidad geográfica del destinatario— para que el `Módulo de Gestión de Rutas` pueda optimizar la consolidación de carga por vehículo.

**Why this priority**: Reduce los tiempos de carga y mejora la eficiencia del algoritmo de rutas. Es invocado por [MOD1-UC-004](./MOD1-UC-004-Preparar-Paquete-Para-Almacenaje.md) una vez que el paquete está en bodega.

**Independent Test**: Con un paquete en `En Clasificación`, verificar que el sistema calcule la zona correcta con el radio configurado (por defecto 5 km), genere la etiqueta digital de zona y actualice el estado a `Clasificado`.

**Acceptance Scenarios**:

1. **Asignación exitosa de zona de destino**
   - **Given** el paquete está en `En Clasificación` con coordenadas GPS válidas y las zonas de destino están configuradas.
   - **When** el sistema calcula la zona de destino y el almacenista confirma.
   - **Then** el sistema asigna la zona, genera la etiqueta digital de zona dentro del sistema y actualiza el estado del paquete a `Clasificado`.

2. **Destino fuera de zonas configuradas**
   - **Given** las coordenadas del paquete no corresponden a ninguna zona configurada.
   - **When** el sistema ejecuta el cálculo.
   - **Then** muestra aviso de `Destino fuera de zonas configuradas` y notifica al Supervisor de Bodega para asignación manual.

### Edge Cases

- **Zona de destino llena**: los paquetes siguientes se redirigen a la zona de desborde configurada o se pausa la clasificación hasta que el Supervisor de Bodega intervenga.
- **Cambio de dirección después de clasificado**: el sistema reclasifica automáticamente si las nuevas coordenadas corresponden a otra zona. Si no hay zona configurada para la nueva dirección, escala al Supervisor de Bodega.
- **Paquete `Frágil` o `Peligroso` en zona no apta**: el sistema bloquea la clasificación y exige seleccionar una zona con la categoría adecuada.

---

## Requirements

### Functional Requirements

- **FR-001**: Agrupar paquetes por proximidad geográfica usando el radio de zona configurado (por defecto 5 km).
- **FR-002**: Generar una etiqueta digital de zona dentro del sistema al confirmar la clasificación. Incluye código de zona, nombre de zona, UUID y código QR. No puede exportarse, descargarse ni imprimirse.
- **FR-003**: Bloquear la clasificación de paquetes `Frágil` o `Peligroso` en zonas no aptas para su categoría.
- **FR-004**: Validar que la zona de destino no haya superado su capacidad máxima antes de confirmar.

### Key Entities

- **Zona de Destino**: nombre, límites geográficos, capacidad máxima y estado de ocupación. Es una clasificación lógica independiente de la Zona de Almacenamiento física.
- **Etiqueta de Zona**: registro digital en el sistema con código de zona, nombre, UUID del paquete y código QR. No exportable ni imprimible.

---

## Success Criteria

- **SC-001**: El 100% de los paquetes clasificados tienen zona de destino asignada antes de avanzar a `Listo para Despacho`.
# Feature Specification: Clasificar Paquete en Zona de Destino (MOD1-UC-004)

**Created**: 2026-02-26

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Agrupación lógica y geográfica (Priority: P2-Alta)

Como Almacenista, necesito agrupar paquetes dentro de la bodega según su zona geográfica de destino.

**Why this priority**: Permite que el Módulo 2 pueda acceder a grupos de paquetes pre-organizados para maximizar la eficiencia en la consolidación de carga y reducir tiempos de carga de vehículos.

**Independent Test**: Puede probarse escaneando un paquete y validando que el algoritmo calcula correctamente su zona basada en coordenadas GPS, finalizando con la impresión de la etiqueta de zona correspondiente.

**Acceptance Scenarios**:

1. **Scenario**: Asignación exitosa a zona de destino
   - **Given**: El paquete está en estado 'En Clasificación', tiene coordenadas GPS y las zonas están configuradas en el sistema.
   - **When**: El almacenista escanea el UUID y el sistema calcula la zona basándose en el radio configurado.
   - **Then**: El almacenista confirma la zona sugerida y el sistema imprime la etiqueta de zona para identificación física.

### Edge Cases

- What happens when: las coordenadas no coinciden con ninguna zona configurada? El sistema muestra un aviso de 'Destino fuera de zonas configuradas' y el supervisor debe asignar el paquete a la zona más apropiada documentando la excepción.
- What happens when: la zona de destino llega al límite de capacidad durante una clasificación masiva? El sistema emite una alerta en tiempo real, redirigiendo los paquetes siguientes a una zona de desborde automáticamente o pausando la clasificación.
- How does system handle: si el cliente cambia la dirección de destino después de la clasificación? El sistema reclasifica automáticamente la zona si las nuevas coordenadas corresponden a otra diferente, y notifica al almacenista para su reubicación física.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST: agrupar paquetes automáticamente según la cercanía geográfica de su destino usando el radio configurado (default: 5 km).
- **FR-002**: System MUST: permitir la impresión o generación de etiquetas de zona para identificar físicamente el grupo de paquetes.
- **FR-003**: System MUST: emitir una alerta si un paquete frágil o peligroso intenta clasificarse en una zona no apta.
- **FR-004**: System MUST: validar que la zona de destino no haya superado su capacidad máxima antes de confirmar la clasificación.
- **NFR-001**: System MUST: lograr una reducción esperada en el tiempo de carga de vehículos del 20% respecto al proceso sin clasificación.

### Key Entities *(include if feature involves data)*

- **Zona de Destino**: Clasificación lógica y física que sirve como entrada para el algoritmo de Consolidación de Carga.
- **Etiqueta de Zona**: Documento físico o digital que identifica el grupo de paquetes asignados a una zona.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los paquetes clasificados deben tener una zona de destino asignada antes de pasar al estado 'Listo para Despacho'.
- **SC-002**: Se debe registrar una reducción del tiempo de carga de vehículos en un 20%, medible en el primer mes post-implementación.
# Feature Specification: Clasificar Paquete por Zona de Destino (MOD1-UC-006)

**Created**: 2026-02-28

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Agrupación lógica y geográfica en bodega (Priority: P2)

Como Almacenista, necesito agrupar los paquetes dentro de la bodega según su zona geográfica de destino para que el Módulo 2 pueda acceder a conjuntos de paquetes pre-organizados y maximizar la eficiencia en la consolidación de carga.

**Why this priority**: Permite reducir los tiempos de carga de vehículos y mejorar la eficiencia del algoritmo de consolidación del Módulo 2. Este caso de uso es invocado por [Preparar Paquete para Almacenaje (MOD1-UC-004)](./MOD1-UC-004-Preparar-Paquete-Para-Almacenaje.md), por lo que depende de que la preparación esté completada.

**Independent Test**: Puede probarse escaneando un paquete en estado `En Clasificación`, validando que el algoritmo calcule correctamente la zona de destino basándose en las coordenadas GPS y el radio configurado (default: 5 km), e imprimiendo la etiqueta de zona correspondiente para verificar que el agrupamiento físico es correcto.

**Acceptance Scenarios**:

1. **Scenario**: Asignación exitosa de zona de destino
   - **Given** el paquete está en estado `En Clasificación`, tiene coordenadas GPS válidas y las zonas de destino están configuradas en el sistema.
   - **When** el almacenista escanea el UUID y el sistema calcula la zona de destino basándose en las coordenadas y el radio_zona_km configurado.
   - **Then** el almacenista confirma la zona sugerida y el sistema imprime o genera la etiqueta de zona para la identificación física del grupo de paquetes.

2. **Scenario**: Destino fuera de zonas configuradas — escalamiento supervisado
   - **Given** las coordenadas GPS del paquete no corresponden a ninguna zona de destino configurada en el sistema.
   - **When** el sistema ejecuta el cálculo de zona.
   - **Then** el sistema muestra un aviso de `Destino fuera de zonas configuradas`, bloquea la asignación automática y notifica al supervisor para que asigne manualmente la zona más apropiada, documentando la excepción.

---

### Edge Cases

- What happens when las coordenadas GPS del paquete no corresponden a ninguna zona configurada? El sistema muestra un aviso de `Destino fuera de zonas configuradas` y el supervisor debe asignar la zona manualmente, dejando constancia de la excepción en el historial del paquete.
- What happens when la zona de destino alcanza su límite de capacidad durante una clasificación masiva? El sistema emite una alerta en tiempo real al almacenista. Los paquetes siguientes para esa zona son redirigidos automáticamente a la zona de desborde configurada o se pausa la clasificación hasta que un supervisor tome acción.
- How does system handle si el cliente cambia la dirección de destino después de que el paquete ya fue clasificado en una zona? El sistema reclasifica automáticamente la zona si las nuevas coordenadas corresponden a una zona diferente y notifica al almacenista para reubicar físicamente el paquete. Si la nueva dirección no tiene zona configurada, se escala al supervisor.
- What happens when un paquete `Frágil` o `Peligroso` intenta clasificarse en una zona no apta para su tipo? El sistema emite una alerta y bloquea la clasificación hasta que se seleccione una zona con la categoría adecuada (Delicada o Alto Riesgo).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST agrupar paquetes automáticamente basándose en la cercanía geográfica de su destino, utilizando el radio_zona_km configurado (valor por defecto: 5 km).
- **FR-002**: System MUST permitir la impresión o generación de etiquetas de zona para identificar físicamente el grupo de paquetes asignados a cada zona.
- **FR-003**: System MUST emitir una alerta si un paquete `Frágil` o `Peligroso` intenta ser clasificado en una zona no apta para su categoría de mercancía.
- **FR-004**: System MUST validar que la zona de destino no haya superado su capacidad máxima configurada antes de confirmar la clasificación del paquete.

### Key Entities *(include if feature involves data)*

- **Zona de Destino**: Clasificación lógica y física en bodega que sirve como entrada para el algoritmo de Consolidación de Carga del Módulo 2. Atributos clave: nombre, límites geográficos, capacidad máxima y estado de ocupación.
- **Etiqueta de Zona**: Documento físico o digital que identifica visualmente el grupo de paquetes asignados a una zona de destino para facilitar su carga al vehículo.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los paquetes clasificados deben tener una zona de destino asignada antes de poder avanzar al estado `Listo para Despacho`.
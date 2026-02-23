# Feature Specification: Preparar paquete para almacenaje

**Created**: 2026-02-23

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Organización Física por Zona de Destino (Priority: P1)

Como Almacenista, quiero organizar los paquetes recibidos en zonas específicas según su destino y tipo de mercancía para optimizar el espacio en bodega y facilitar su despacho posterior.

**Why this priority**: Es el paso crítico para eliminar "puntos ciegos" en la bodega. Sin una clasificación correcta por zona, el algoritmo de consolidación de carga del Módulo 2 no podría agrupar los paquetes por cercanía geográfica para optimizar combustible.

**Independent Test**: Se puede testear escaneando un paquete en estado 'Recibido en Sede' y verificando que el sistema permita asignarle una ubicación de estantería, cambiando su estado a "En Clasificación".

**Acceptance Scenarios**:

1. **Scenario**: Clasificación exitosa por zona geográfica.
* **Given** un paquete con estado 'Recibido en Sede' que ya cuenta con coordenadas GPS validadas.
* **When** el almacenista escanea el UUID y confirma su ubicación en la zona de destino correspondiente.
* **Then** el sistema actualiza el estado del paquete a 'En Clasificación' y zona geográfica.

2. **Scenario**: Validación de mercancía especial (Frágil/Peligrosa).
* **Given** un paquete registrado con el atributo físico 'Frágil' o 'Peligrosos'.
* **When** el almacenista escanea el UUID, confirma que el estado del paquete sea 'Frágil' o 'Peligroso' y su ubicacion en la zona de destino correspondiente.
* **Then** el sistema emite una alerta de manejo especial y sugiere una ubicación de almacenamiento segura para evitar averías.

### Edge Cases

* ¿Qué sucede si la zona física de destino está al tope de su capacidad? 
  Se actualiza el estado de la zona física a 'Saturado' y se cambia la ubicación del paquete a una zona física de categoria 'Zona de Retención'.
  
* ¿Cómo maneja el sistema un paquete cuyo peso físico detectado en bodega difiere significativamente del registrado en la admisión?
  Se actualiza el estado del paquete cuyo peso físico sea significativamente diferente al inicialmente registrado, pasara a un estado 'Fuera de Tolerancia'

## Requirements *(mandatory)*

### Functional Requirements

* **FR-001**: El sistema DEBE permitir al almacenista asignar una ubicación física (estante/zona) vinculada al UUID del paquete.
* **FR-002**: El sistema DEBE actualizar automáticamente el estado del paquete a los estados posibles de acuerdo a las condiciones ('En Clasificación', 'Fuera de Tolerancia', 'Frágil', 'Peligroso') una vez se confirma su ubicación.
* **FR-003**: El sistema DEBE restringir la clasificación de paquetes que no tengan dirección exacta o coordenadas GPS completas.

### Key Entities

* **UUID**: Identificador único del paquete que garantiza la trazabilidad desde el ingreso.
* **Paquete**: Entidad central que contiene el UUID, fechas de ingreso, destinos, dimensiones, peso y estados.
* **Zona de Almacenamiento**: Espacio físico en bodega donde se agrupan los paquetes por cercanía geográfica para facilitar la labor del Módulo 2. Este espacio físico debe tener atributos propios como UUID, nombre, capacidad, estado, categoria.

## Success Criteria *(mandatory)*

### Measurable Outcomes

* **SC-001**: El 100% de los paquetes en estado "En Clasificación" deben tener asignada una zona de destino compatible con la lógica del Módulo 2. //PENDIENTE POR ESPECIFICAR  QUE una vez finalizada la clasificación, el 100% de los paquetes deben tener un estado diferente a 'En Clasificacion'
* **SC-002**: Reducción del tiempo de búsqueda de paquetes durante el despacho al tener ubicaciones físicas registradas en el sistema, de tal manera que el tiempo entre la consulta y la respuesta del sitema no dure más de 45 segundos.

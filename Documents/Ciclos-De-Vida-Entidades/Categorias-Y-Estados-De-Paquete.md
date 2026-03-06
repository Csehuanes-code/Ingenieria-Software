# Categorías y Clasificaciones de Paquete — Módulo de Gestión de Paquetes (MOD1)

**Versión**: 1.0  
**Fecha**: 2026-02-28  
**Alcance**: Define todas las categorías y clasificaciones posibles de un paquete, las condiciones que debe cumplir para ingresar a cada una, las restricciones de transición de estado asociadas y la justificación operativa de cada clasificación. Este documento es complementario al [Ciclo de Vida de un Paquete](./MOD1-CICLO-DE-VIDA-PAQUETE.md).

---

## 1. Introducción

Un paquete en el sistema no tiene una única categoría: puede tener múltiples clasificaciones simultáneas e independientes que afectan distintos aspectos de su manejo, tarificación, almacenamiento y transporte. Estas clasificaciones se asignan durante la admisión y pueden ser revisadas durante el procesamiento en bodega.

Las clasificaciones están organizadas en cuatro dimensiones:

| Dimensión | Clasificaciones posibles | Se asigna en |
|---|---|---|
| Tipo de Mercancía | `Estándar`, `Frágil`, `Peligroso` | MOD1-UC-002 |
| Categoría de Carga | `Normal`, `Carga Especial` | MOD1-UC-002 |
| Prioridad de Entrega | `Estándar`, `Urgente` | MOD1-UC-001 |
| Estado Operativo | `Borrador`, `Recibido en Sede`, `En Clasificación`, `Clasificado`, `En Carga`, `Listo para Despacho`, `En Tránsito`, `Entregado`, `Dañado`, `Extraviado`, `Devolución`, `Fuera de Tolerancia`, `Excepción de Ruta`, `En Espera de Instrucción`, `Pendiente Sincronización Contable`, `Sincronizado Contablemente` | Varios |

---

## 2. Dimensión: Tipo de Mercancía

### 2.1 Estándar

**Condiciones de clasificación**: El paquete no cumple ningún criterio de Frágil ni Peligroso. Puede manipularse sin equipos especiales ni protecciones adicionales. No requiere documentación de seguridad.

**Características operativas**:
- Puede almacenarse en cualquier zona de almacenamiento, incluyendo categoría `Normal`.
- No genera recargo tarifario por tipo de mercancía.
- No requiere autorización de supervisor para su registro.
- Puede ser asignado a cualquier vehículo con capacidad de peso y volumen suficiente.

**Por qué es importante especificarlo**: Establece la línea base del flujo operativo estándar y permite al sistema identificar los paquetes que no requieren tratamiento especial, optimizando tiempos de procesamiento.

**Restricciones de transición de estado**: Sin restricciones adicionales a las del ciclo de vida general.

---

### 2.2 Frágil

**Condiciones de clasificación**: El paquete contiene objetos susceptibles de romperse, deformarse o deteriorarse por impacto, vibración, presión o condiciones ambientales adversas. Lo declara el remitente al momento de la admisión. Ejemplos: vidrio, cerámica, electrónicos sin blindaje industrial, instrumentos musicales, obras de arte, espejos, porcelana.

**Características operativas**:
- Solo puede almacenarse en zonas de almacenamiento de categoría `Delicada`.
- Genera recargo tarifario `tarifa_recargo_fragil` sobre el precio de envío.
- La etiqueta física del paquete debe incluir el símbolo estándar de manejo frágil ("Copa rota") y la indicación "Este lado arriba".
- Los vehículos asignados deben tener condiciones de amortiguación aptas para frágiles (configuración en el `Módulo de Gestión de Rutas`).
- No puede apilarse sobre otros paquetes en el vehículo.

**Por qué es importante especificarlo**: Evita daños durante el almacenamiento y transporte, reduce novedades de tipo `Dañado` y protege a la empresa de indemnizaciones por manejo inadecuado declarado previamente por el remitente.

**Restricciones de transición de estado**:
- No puede avanzar a `En Clasificación` si la zona de almacenamiento asignada no es de categoría `Delicada`.
- Si durante la inspección en bodega se detecta daño, el paquete pasa directamente a novedad tipo `Dañado` sin importar el estado actual.

---

### 2.3 Peligroso

**Condiciones de clasificación**: El paquete contiene sustancias o materiales que representan riesgo de incendio, explosión, toxicidad, corrosión, reactividad o daño biológico. Requiere declaración formal del remitente y presentación de la hoja de datos de seguridad (MSDS o SDS). Ejemplos: baterías de litio, ácidos, solventes inflamables, aerosoles bajo presión, materiales radiactivos de baja actividad, desinfectantes concentrados.

**Características operativas**:
- Solo puede almacenarse en zonas de categoría `Alto Riesgo`.
- Está sujeto a límites estrictos de peso y dimensiones por operación, configurables por el Administrador del Sistema.
- Genera el recargo tarifario más alto: `tarifa_recargo_peligroso`.
- Requiere documentación adjunta (MSDS) en el registro del paquete.
- Los vehículos asignados deben estar habilitados para transporte de mercancía peligrosa (configuración en el `Módulo de Gestión de Rutas`).
- Si las dimensiones o el peso exceden los límites configurados, se requiere autorización explícita del Supervisor de Admisión (autenticación en el formulario) para continuar.

**Por qué es importante especificarlo**: Cumplimiento legal y de seguridad. El manejo inadecuado de mercancía peligrosa puede causar accidentes, daños a terceros y responsabilidad legal para la empresa. Es la categoría de más alto riesgo operativo del sistema.

**Restricciones de transición de estado**:
- No puede avanzar a `En Clasificación` si la zona de almacenamiento no es de categoría `Alto Riesgo`.
- No puede asignarse a vehículos no habilitados para mercancía peligrosa.
- Ante cualquier señal de derrame, fuga o daño del empaque durante cualquier etapa, el paquete pasa inmediatamente a novedad tipo `Dañado` y se activa el protocolo de seguridad de la sede.

---

## 3. Dimensión: Categoría de Carga

### 3.1 Normal

**Condiciones de clasificación**: `peso_kg` ≤ 50 kg **Y** `volumen_m3` ≤ 0.5 m³. Puede manipularse manualmente por una sola persona sin equipos especiales.

**Características operativas**:
- Flujo estándar de procesamiento y carga.
- Sin recargo adicional por manipulación.
- Puede asignarse a cualquier vehículo con capacidad suficiente.

**Por qué es importante especificarlo**: Define el umbral de operación manual segura para los trabajadores, en cumplimiento de normas ergonómicas y de seguridad laboral.

**Restricciones de transición de estado**: Sin restricciones adicionales.

---

### 3.2 Carga Especial

**Condiciones de clasificación**: `peso_kg` > 50 kg **O** `volumen_m3` > 0.5 m³. Requiere equipos de carga (montacargas, transpaleta, grúa de bodega o banda transportadora) para su manejo seguro.

**Características operativas**:
- Genera recargo tarifario `tarifa_recargo_carga_especial` sobre el precio de envío.
- El sistema emite la alerta `Carga Especial` durante el pesaje y solicita confirmación explícita del empleado.
- Restringe los vehículos elegibles para el transporte (solo vehículos con rampa de carga o plataforma habilitados).
- El Coordinador de Despacho debe verificar la disponibilidad de equipo de carga antes de iniciar la carga al vehículo.

**Por qué es importante especificarlo**: Previene lesiones laborales, daños al paquete y a la infraestructura. Garantiza que la tarifa cobrada cubra el costo real del manejo especial.

**Restricciones de transición de estado**:
- No puede avanzar a `En Carga` si no hay equipo de carga disponible en el andén.
- Si el equipo de carga no está disponible al momento de la carga, el paquete permanece en `Clasificado` hasta que el equipo esté habilitado.

---

## 4. Dimensión: Prioridad de Entrega

### 4.1 Estándar

**Condiciones de clasificación**: Todos los paquetes por defecto. El remitente no pagó tarifa de urgencia.

**Características operativas**:
- El tiempo estimado de entrega sigue la planificación normal de rutas del `Módulo de Gestión de Rutas`.
- Sin recargo por prioridad.
- Se asigna al primer vehículo disponible según la optimización de ruta normal.

**Por qué es importante especificarlo**: Es el estado base que permite distinguir claramente los paquetes que requieren tratamiento de urgencia.

**Restricciones de transición de estado**: Sin restricciones adicionales.

---

### 4.2 Urgente

**Condiciones de clasificación**: El remitente paga explícitamente la tarifa de urgencia (`tarifa_recargo_urgente`) durante la admisión **Y** el destino tiene cobertura de entrega express disponible (validado por el `Módulo de Gestión de Rutas` en la respuesta `ruta_asignada`). Si el destino no tiene cobertura express, el sistema informa al empleado y el paquete se clasifica como `Estándar` con devolución del recargo.

**Características operativas**:
- El payload enviado al `Módulo de Gestión de Rutas` incluye el campo `prioridad: "Urgente"`.
- M2 asigna el primer vehículo disponible en la zona, con prioridad sobre los paquetes `Estándar`.
- Se garantiza despacho en el mismo día hábil si la admisión ocurre antes de las 12:00 hrs.
- Si la admisión ocurre después de las 12:00 hrs, se garantiza despacho en el primer turno del siguiente día hábil.
- El tiempo estimado de entrega mostrado al cliente refleja la prioridad urgente.
- Los paquetes `Urgente` tienen prioridad en la cola de carga del Coordinador de Despacho.

**Por qué es importante especificarlo**: Permite al sistema y al `Módulo de Gestión de Rutas` distinguir entre paquetes que requieren asignación inmediata y los que siguen el flujo regular, evitando confusiones operativas y garantizando el nivel de servicio comprometido con el cliente.

**Restricciones de transición de estado**:
- Si el paquete no puede ser despachado en el mismo día hábil comprometido (por saturación de flota u otras causas), el sistema genera una alerta al Supervisor de Admisión para que notifique al cliente y aplique las compensaciones correspondientes.

---

## 5. Dimensión: Estado Operativo

Los estados operativos representan la posición del paquete en el ciclo de vida del `Módulo de Gestión de Paquetes`. Cada estado tiene condiciones de entrada, actores responsables y restricciones de transición.

### 5.1 Borrador

**Condiciones de entrada**: El empleado abre el formulario de registro. UUID generado provisionalmente. Pesaje aún no completado.

**Restricciones**: El sistema no emite ningún evento externo desde este estado. Ningún usuario puede modificar el UUID ni el `timestamp_creacion`. El sistema elimina automáticamente los registros en este estado con más de 30 minutos de inactividad (job de limpieza FR-011 de MOD1-UC-001).

**Transiciones permitidas**: `Borrador` → `Recibido en Sede` (al confirmar registro con pesaje exitoso) | `Borrador` → ELIMINADO (por job de limpieza o cancelación del empleado).

---

### 5.2 Recibido en Sede

**Condiciones de entrada**: Registro confirmado con todos los campos obligatorios completos, pesaje exitoso completado y `gps_estado = Resuelto`. La solicitud de ruta al `Módulo de Gestión de Rutas` fue emitida (o está pendiente si `gps_estado = Pendiente GPS`).

**Restricciones**: `uuid`, `timestamp_ingreso` e `id_sede` son inmutables desde este estado. El paquete no puede avanzar a `En Clasificación` mientras `gps_estado = Pendiente GPS`.

**Transiciones permitidas**: `Recibido en Sede` → `En Clasificación` (al confirmar zona de almacenamiento en MOD1-UC-004) | `Recibido en Sede` → `Fuera de Tolerancia` (diferencia de peso > 10% en bodega).

---

### 5.3 En Clasificación

**Condiciones de entrada**: Zona de almacenamiento física asignada por el Almacenista en MOD1-UC-004. Los contadores de capacidad de la zona fueron actualizados.

**Restricciones**: El paquete no puede avanzar a `Clasificado` si la zona de destino no ha sido asignada (MOD1-UC-006 pendiente). No puede avanzar si tiene `Fuera de Tolerancia` activo.

**Transiciones permitidas**: `En Clasificación` → `Clasificado` (al completar MOD1-UC-006) | `En Clasificación` → `Fuera de Tolerancia` (por inconsistencia detectada).

---

### 5.4 Clasificado

**Condiciones de entrada**: Zona de destino lógica asignada en MOD1-UC-006. El paquete tiene tanto zona de almacenamiento física como zona de destino lógica asignadas. El paquete está listo para ser embalado y confirmado para el andén.

**Restricciones**: No puede avanzar a `Listo para Despacho` sin pasar por MOD1-UC-005 (confirmación del almacenista de que el embalaje está listo). No puede avanzar a `En Carga` sin `id_ruta` e `id_transportador` disponibles en la entidad Solicitud de Ruta.

**Transiciones permitidas**: `Clasificado` → `Listo para Despacho` (MOD1-UC-005) | `Clasificado` → `En Carga` (MOD1-UC-007, cuando el Coordinador escanea el paquete para cargarlo) | `Clasificado` → `Dañado` (si se detecta daño durante inspección).

---

### 5.5 En Carga

**Condiciones de entrada**: El Coordinador de Despacho escanea el UUID del paquete al recogerlo de la zona de almacenamiento para cargarlo al vehículo. El paquete debe tener `id_ruta` e `id_transportador` disponibles.

**Restricciones**: El paquete está temporalmente en manos del Coordinador de Despacho. Si transcurren más de 60 minutos en este estado sin actividad, se genera alerta al Supervisor de Bodega. No puede haber dos coordinadores asignados simultáneamente al mismo paquete (control FIFO).

**Transiciones permitidas**: `En Carga` → `Listo para Despacho` (al confirmar ubicación en vehículo correcto) | `En Carga` → `Clasificado` (si se detecta daño o discrepancia durante la carga y se cancela la operación) | `En Carga` → `Dañado` (si se detecta daño durante la manipulación).

---

### 5.6 Listo para Despacho

**Condiciones de entrada**: El paquete fue verificado por el Coordinador de Despacho: vehículo, zona de destino y ruta correctos. El paquete está físicamente ubicado en el vehículo.

**Restricciones**: Ningún usuario puede revertir este estado de forma unilateral. Cualquier cambio posterior requiere autorización del Supervisor de Bodega. El Almacenista no puede transicionar un paquete que ya está en este estado. El Empleado de Envío y Recepción no tiene permiso sobre este estado.

**Transiciones permitidas**: `Listo para Despacho` → `En Tránsito` (cuando M2 confirma el despacho del vehículo) | `Listo para Despacho` → `Dañado` / `Extraviado` (por novedad detectada antes de que el vehículo salga).

---

### 5.7 En Tránsito

**Condiciones de entrada**: El `Módulo de Gestión de Rutas` emitió el estado `En Tránsito` tras recibir la notificación `vehiculo_listo`. El paquete está en el vehículo en ruta hacia el destinatario.

**Restricciones**: Ningún actor del `Módulo de Gestión de Paquetes` puede modificar el estado directamente en este punto. Solo el `Módulo de Gestión de Rutas` puede transicionar desde `En Tránsito`.

**Transiciones permitidas**: `En Tránsito` → `Entregado` (M2) | `En Tránsito` → `Dañado` / `Extraviado` / `Devolución` (M2 reporta novedad).

---

### 5.8 Entregado

**Condiciones de entrada**: El `Módulo de Gestión de Rutas` confirma la entrega exitosa al destinatario, con firma digital (POD) obtenida.

**Restricciones**: Estado terminal positivo. No puede revertirse. El sistema invoca automáticamente MOD1-UC-009 para notificar al `Módulo de Gestión de Finanzas`.

**Transiciones permitidas**: `Entregado` → `Pendiente Sincronización Contable` (automático, vía MOD1-UC-009).

---

### 5.9 Dañado

**Condiciones de entrada**: El Controlador de Novedades registra avería física con evidencia adjunta válida (JPEG/PNG/MP4, ≤ 10 MB) en MOD1-UC-008. Puede ocurrir desde `En Clasificación` en adelante.

**Restricciones**: Requiere evidencia obligatoria para registrar la novedad. El Controlador de Novedades no puede cambiar el tipo de novedad una vez registrado sin autorización del Supervisor de Novedades. Se notifica automáticamente al remitente y destinatario.

**Transiciones permitidas**: `Dañado` → `Pendiente Sincronización Contable` (automático, vía MOD1-UC-009).

---

### 5.10 Extraviado

**Condiciones de entrada**: El Controlador de Novedades registra el extravío del paquete en MOD1-UC-008. Puede ocurrir desde `En Clasificación` en adelante.

**Restricciones**: El estado puede actualizarse con nueva información (evidencias adicionales, descripción técnica) pero el tipo de novedad no puede cambiarse sin autorización del Supervisor de Novedades. Se notifica automáticamente al remitente y destinatario.

**Transiciones permitidas**: `Extraviado` → `Pendiente Sincronización Contable` (automático, vía MOD1-UC-009).

---

### 5.11 Devolución

**Condiciones de entrada**: El `Módulo de Gestión de Rutas` retorna el paquete a la sede (dirección errónea, cliente no encontrado, rechazo de la entrega) y el Controlador de Novedades registra el re-ingreso en MOD1-UC-008.

**Restricciones**: El paquete debe ser inspeccionado físicamente antes de definir su próximo estado. Se notifica automáticamente al remitente y destinatario.

**Estado siguiente depende del análisis post-devolución**:

| Resultado del análisis | Estado resultante |
|---|---|
| Paquete en buen estado | `En Clasificación` (por defecto) |
| Daño detectado durante la devolución | `Dañado` |
| Requiere instrucción del remitente | `En Espera de Instrucción` |
| Contenido parcialmente extraviado | `Extraviado` |

---

### 5.12 Fuera de Tolerancia

**Condiciones de entrada**: El sistema detecta una inconsistencia bloqueante durante el procesamiento en bodega. Subtipos:
- `Fuera de Tolerancia — Peso`: diferencia > 10% entre `peso_kg` registrado en admisión y peso físico observado en bodega.
- `Fuera de Tolerancia — Sin GPS`: `gps_estado = Pendiente GPS` al momento de intentar asignar zona de almacenamiento.

**Restricciones**: El paquete no puede avanzar a ningún estado operativo siguiente hasta que la inconsistencia sea resuelta y aprobada por el Supervisor de Bodega. No se emiten eventos externos mientras este estado esté activo.

**Transiciones permitidas**: `Fuera de Tolerancia` → `Recibido en Sede` (al resolver y aprobar la inconsistencia, el paquete vuelve al flujo).

---

### 5.13 Excepción de Ruta

**Condiciones de entrada**: El `Módulo de Gestión de Rutas` rechaza la solicitud de ruta por cobertura no disponible o campo inválido en el payload.

**Restricciones**: El Empleado de Envío y Recepción debe corregir el dato que causó el rechazo (dirección, coordenadas, etc.) bajo supervisión del Supervisor de Admisión. No se puede re-emitir la solicitud de ruta sin la corrección aprobada.

**Transiciones permitidas**: `Excepción de Ruta` → `Recibido en Sede` (al corregir el dato y re-emitir exitosamente la solicitud de ruta).

---

### 5.14 En Espera de Instrucción

**Condiciones de entrada**: El análisis post-devolución determina que el paquete requiere instrucción explícita del remitente antes de definir el próximo paso (ej. re-despacho con nueva dirección, reclamar el paquete en sede, autorizar destrucción).

**Restricciones**: El sistema envía una notificación al remitente solicitando instrucción. El paquete permanece en este estado hasta recibir respuesta. Si no se recibe respuesta en 5 días hábiles, el Controlador de Novedades escala al Supervisor de Novedades para definir el siguiente paso.

**Transiciones permitidas**: `En Espera de Instrucción` → `En Clasificación` (remitente solicita re-despacho) | `En Espera de Instrucción` → `Devolución cerrada` (remitente recoge el paquete en sede) | `En Espera de Instrucción` → `Extraviado` (si el contenido fue parcialmente perdido y no hay re-despacho viable).

---

### 5.15 Pendiente Sincronización Contable

**Condiciones de entrada**: MOD1-UC-009 fue invocado pero el `Módulo de Gestión de Finanzas` aún no respondió con ACK.

**Restricciones**: El sistema reintenta automáticamente la notificación cada 5 minutos. No se realizan operaciones adicionales sobre el paquete mientras el ACK esté pendiente.

**Transiciones permitidas**: `Pendiente Sincronización Contable` → `Sincronizado Contablemente` (al recibir ACK de M3).

---

### 5.16 Sincronizado Contablemente

**Condiciones de entrada**: El `Módulo de Gestión de Finanzas` confirmó (ACK) la recepción del estado final del paquete y ejecutó las acciones financieras correspondientes.

**Restricciones**: Estado terminal. Es el cierre definitivo del ciclo de vida del paquete en el `Módulo de Gestión de Paquetes`. Ningún actor puede modificar ningún atributo del paquete desde este estado. El registro del paquete queda archivado de forma permanente e inmutable.

**Transiciones permitidas**: Ninguna. Fin del ciclo de vida.

---

## 6. Matriz de Restricciones de Transición por Rol

| Estado actual | Almacenista | Coordinador de Despacho | Controlador de Novedades | Empleado de Envío y Recepción | Sistema (automático) |
|---|---|---|---|---|---|
| `Borrador` | ✗ | ✗ | ✗ | Confirmar → `Recibido en Sede` | Eliminar tras 30 min |
| `Recibido en Sede` | Asignar zona → `En Clasificación` | ✗ | ✗ | Resolver GPS | Emitir `solicitar_ruta` |
| `En Clasificación` | Clasificar zona → `Clasificado` | ✗ | Registrar novedad | ✗ | ✗ |
| `Clasificado` | Confirmar embalaje → `Listo para Despacho` | Escanear → `En Carga` | Registrar novedad | ✗ | ✗ |
| `En Carga` | ✗ | Confirmar en vehículo → `Listo para Despacho` | Registrar novedad | ✗ | Alerta tras 60 min |
| `Listo para Despacho` | ✗ (solo Supervisor puede revertir) | Emitir `vehiculo_listo` | Registrar novedad | ✗ | Notificar M2 |
| `En Tránsito` | ✗ | ✗ | ✗ | ✗ | Recibir evento de M2 |
| `Entregado` | ✗ | ✗ | ✗ | ✗ | Invocar MOD1-UC-009 |
| `Dañado` / `Extraviado` | ✗ | ✗ | Agregar evidencia | ✗ | Invocar MOD1-UC-009 |
| `Devolución` | ✗ | ✗ | Analizar → estado resultado | ✗ | Invocar MOD1-UC-009 |
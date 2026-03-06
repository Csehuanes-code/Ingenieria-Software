# Responsabilidades y Compromisos — Cliente y Servicio de Paquetería (MOD1)

**Versión**: 1.0  
**Fecha**: 2026-02-28  
**Alcance**: Define los compromisos del servicio de paquetería hacia el cliente (remitente y destinatario) y las obligaciones del cliente hacia el servicio. Este documento está basado en los Requisitos Funcionales y Criterios de Éxito de las especificaciones MOD1-UC-001 a MOD1-UC-009.

---

## 1. Partes Involucradas

**Servicio de Paquetería**: La empresa operadora del `Módulo de Gestión de Paquetes`, responsable de la admisión, procesamiento, almacenamiento, despacho y seguimiento de los paquetes.

**Remitente**: La persona natural o jurídica que entrega el paquete en la sede para su envío. Proporciona los datos del envío y acepta las condiciones del servicio al momento de la admisión.

**Destinatario**: La persona natural o jurídica que recibe el paquete en la dirección de destino.

---

## 2. Lo que el Servicio de Paquetería le Asegura al Cliente

### 2.1 Durante la Admisión

| Compromiso | Detalle | Referencia |
|---|---|---|
| **Identidad única del envío** | El sistema genera un UUID único e irrepetible para cada paquete. El remitente recibe una etiqueta física (formato ZPL con código QR y código de barras) que identifica su paquete de forma inequívoca. | MOD1-UC-001 FR-002, FR-006 |
| **Transparencia en el precio de envío** | El precio de envío calculado se muestra al remitente antes de confirmar el registro. El desglose incluye tarifa base, peso facturable, distancia, tipo de mercancía, categoría de carga y prioridad. Una vez confirmado el registro, el precio es inmutable. | MOD1-UC-002 FR-007 |
| **Confirmación de ruta y tiempo estimado** | Inmediatamente tras la admisión exitosa, el sistema solicita la asignación de ruta al `Módulo de Gestión de Rutas`. Si la respuesta llega antes de que el cliente abandone el local, se imprime en el comprobante la fecha estimada de entrega. Si la respuesta está pendiente, el comprobante incluye la leyenda `Fecha de entrega sujeta a confirmación`. | MOD1-UC-003 FR-003 |
| **Prioridad Urgente garantizada** | Para paquetes con tarifa de urgencia pagada, se garantiza despacho en el mismo día hábil si la admisión ocurre antes de las 12:00 hrs, o en el primer turno del siguiente día hábil si ocurre después de las 12:00 hrs, siempre que el destino tenga cobertura express. | MOD1-CATEGORIAS §4.2 |
| **Protección de datos** | Los datos del remitente y destinatario (documento, nombre, teléfono, email, dirección) son capturados exclusivamente para la gestión del envío. No se comparten con terceros fuera del flujo operativo del servicio. | MOD1-UC-001 FR-009 |

### 2.2 Durante el Procesamiento en Bodega

| Compromiso | Detalle | Referencia |
|---|---|---|
| **Manejo diferenciado por tipo de mercancía** | Los paquetes clasificados como `Frágil` se almacenan exclusivamente en zonas de categoría `Delicada` y se manipulan con precaución especial. Los paquetes `Peligrosos` se almacenan en zonas `Alto Riesgo` y son manejados según los protocolos de seguridad aplicables. | MOD1-UC-004 FR-004 |
| **Trazabilidad completa** | Cada transición de estado del paquete queda registrada en el Historial de Estados con `timestamp` UTC e identificación del responsable. El remitente puede consultar el historial completo del paquete. | MOD1-UC-005 FR-005 |
| **Notificación proactiva ante novedades** | Ante cualquier novedad (daño, extravío, devolución), el sistema notifica automáticamente al remitente (al teléfono registrado) y al destinatario (al teléfono y email registrados) informando el tipo de novedad y el estado actual del paquete. | MOD1-UC-008 FR-005 |

### 2.3 Durante el Tránsito y la Entrega

| Compromiso | Detalle | Referencia |
|---|---|---|
| **Custodia hasta la entrega** | El servicio mantiene la cadena de custodia desde la admisión en sede hasta la confirmación de entrega al destinatario. El estado `En Tránsito` es gestionado por el `Módulo de Gestión de Rutas`. | MOD1-UC-007, ciclo de vida |
| **Prueba de entrega (POD)** | Al completarse la entrega, el sistema registra la firma digital del destinatario como prueba de entrega (`url_pod`). Este dato queda asociado permanentemente al UUID del paquete. | MOD1-UC-009 FR-002 |

### 2.4 Ante Incidencias

| Compromiso | Detalle | Referencia |
|---|---|---|
| **Gestión de daños** | Si el paquete llega dañado y la causa es atribuible al manejo durante el transporte, el `Módulo de Gestión de Finanzas` aplica el descuento al transportador y/o el cobro de la póliza de seguro por el `valor_declarado` del paquete. | MOD1-UC-008, MOD1-UC-009 |
| **Indemnización por extravío** | Si el paquete se extravía bajo custodia del servicio, se activa la indemnización basada en el `valor_declarado` declarado por el remitente al momento de la admisión, a través del `Módulo de Gestión de Finanzas`. | MOD1-UC-009 §Estado Final |
| **Gestión de devoluciones** | Si el paquete no puede ser entregado (dirección incorrecta, destinatario no encontrado, rechazo de la entrega), el paquete es retornado a sede y el remitente es notificado. El ajuste financiero por logística inversa se ejecuta según la matriz de costos de retorno del `Módulo de Gestión de Finanzas`. | MOD1-UC-008 FR-003, FR-005 |
| **Comunicación de nuevos tiempos estimados** | Si los datos físicos del paquete son corregidos durante el procesamiento en bodega (diferencia de peso > 10%), el sistema re-emite la solicitud de ruta con los datos corregidos y notifica al remitente los nuevos tiempos estimados de entrega. | MOD1-UC-003 §Reintentos |

---

## 3. Lo que el Servicio de Paquetería Espera del Cliente (Remitente)

### 3.1 Información veraz y completa

| Obligación | Detalle | Consecuencia del incumplimiento |
|---|---|---|
| **Datos de identificación** | Proporcionar documento (tipo y número), nombre completo y teléfono. Los datos deben ser verídicos y actuales. | Si los datos son incorrectos, las notificaciones no llegarán al remitente y las gestiones de novedad o devolución no podrán ejecutarse correctamente. |
| **Datos del destinatario** | Proporcionar documento, nombre completo, teléfono, email y dirección de entrega completa y precisa. | Si la dirección es incorrecta o incompleta, el `Módulo de Gestión de Rutas` puede rechazar la solicitud de ruta (`Excepción de Ruta`) o el paquete puede ser devuelto con costo de logística inversa para el remitente. |
| **Declaración del contenido** | Declarar correctamente si el paquete contiene mercancía `Frágil` o `Peligrosa`. | Si el contenido real difiere de la declaración y causa daños (a otros paquetes, al vehículo o a personas), la responsabilidad recae sobre el remitente. El servicio no responde por daños causados por mercancía peligrosa no declarada. |
| **Valor declarado veraz** | El `valor_declarado` debe corresponder al valor comercial real del contenido. | Un valor subdeclarado limita el monto máximo de indemnización en caso de extravío o daño. Un valor sobredeclarado puede implicar recargos no justificados y acciones legales. |

### 3.2 Empaque adecuado

| Obligación | Detalle | Consecuencia del incumplimiento |
|---|---|---|
| **Empaque apropiado para el tipo de mercancía** | Los paquetes `Frágiles` deben llegar con empaque de protección (burbuja, espuma, relleno) que evite movimiento interno del contenido. Los paquetes `Peligrosos` deben cumplir el empaque reglamentario según la hoja de datos de seguridad (MSDS). | Los daños causados por empaque insuficiente no son cubiertos por la póliza de seguro del servicio. |
| **Presentación física al momento de la admisión** | El paquete debe presentarse físicamente en la sede para su pesaje y medición. No se aceptan declaraciones de dimensiones o peso sin verificación presencial. | Sin el pesaje físico completado (MOD1-UC-002), el registro no puede confirmarse y el paquete no ingresa al sistema. |

### 3.3 Documentación para mercancía peligrosa

| Obligación | Detalle | Consecuencia del incumplimiento |
|---|---|---|
| **Hoja de datos de seguridad (MSDS/SDS)** | Todo paquete clasificado como `Peligroso` debe estar acompañado de la hoja MSDS o SDS del material, presentada al Empleado de Envío y Recepción en el momento de la admisión. | Sin la documentación requerida, el sistema no permite confirmar el registro del paquete `Peligroso`. El paquete no puede ingresar al flujo operativo. |

### 3.4 Disponibilidad para comunicación

| Obligación | Detalle | Consecuencia del incumplimiento |
|---|---|---|
| **Teléfono activo** | El teléfono del remitente debe estar activo y disponible para recibir mensajes de notificación sobre el estado del paquete, novedades y devoluciones. | Si el remitente no responde a la notificación de `En Espera de Instrucción` en 5 días hábiles, el Supervisor de Novedades define unilateralmente el siguiente paso para el paquete. |

---

## 4. Limitaciones del Servicio

| Limitación | Detalle |
|---|---|
| **Cobertura geográfica** | El servicio opera únicamente en las zonas de cobertura configuradas en el `Módulo de Gestión de Rutas`. Si el destino está fuera de cobertura, el paquete entra en estado `Excepción de Ruta` y el remitente debe corregir la dirección o desistir del envío. |
| **Límites de peso operativo** | El sistema no admite paquetes con `peso_kg` > 4.500 kg. Pesos mayores requieren gestión manual fuera del sistema estándar. |
| **Responsabilidad por mercancía no declarada** | El servicio no asume responsabilidad por daños o pérdidas de mercancía cuyo contenido real difiera de la declaración del remitente. |
| **Responsabilidad limitada al valor declarado** | El monto máximo de indemnización ante extravío o daño está limitado al `valor_declarado` registrado en la admisión. El servicio no reconoce valores superiores a los declarados. |
| **Tiempos de entrega sujetos a condiciones operativas** | Los tiempos estimados de entrega son proporcionados por el `Módulo de Gestión de Rutas` y están sujetos a variaciones por condiciones de tráfico, clima, disponibilidad de flota u otras circunstancias fuera del control directo del servicio. |

---

## 5. Proceso de Reclamaciones

| Situación | Cómo proceder |
|---|---|
| **Paquete dañado** | El destinatario o remitente debe reportar el daño al Controlador de Novedades en sede, quien registra la novedad con evidencia fotográfica en MOD1-UC-008. El `Módulo de Gestión de Finanzas` procesa la compensación. |
| **Paquete extraviado** | El remitente debe contactar al Controlador de Novedades en sede para iniciar el registro de extravío. El sistema activa la indemnización basada en el `valor_declarado`. |
| **Paquete devuelto** | El remitente recibe notificación automática. Debe responder con instrucciones (re-despacho, recogida en sede, etc.) dentro de los 5 días hábiles siguientes a la notificación. |
| **Discrepancia en precio de envío** | El precio de envío calculado se muestra antes de confirmar el registro y queda inmutable una vez confirmado. Las reclamaciones sobre el precio calculado deben realizarse antes de confirmar el registro. |
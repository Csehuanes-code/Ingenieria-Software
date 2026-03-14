# Responsabilidades — Cliente y Servicio de Paquetería (MOD1)

**Versión**: 2.0 | **Fecha**: 2026-02-28

---

## 1. Lo que el servicio garantiza al cliente

### Durante la admisión

| Compromiso | Detalle |
|---|---|
| **Identidad única del envío** | Se genera un UUID irrepetible por paquete. El cliente recibe una etiqueta digital en el sistema con código QR y código de barras. |
| **Precio transparente** | El precio de envío se muestra al cliente antes de confirmar el registro, con su desglose completo. Una vez confirmado, es inmutable. |
| **Tiempo estimado de entrega** | Al completar la admisión se solicita ruta al `Módulo de Gestión de Rutas`. Si hay respuesta antes de que el cliente se retire, la fecha estimada queda registrada. Si está pendiente, se indica explícitamente. |
| **Protección de datos personales** | Los datos de remitente y destinatario se usan exclusivamente para la gestión del envío. |

### Durante el procesamiento en bodega

| Compromiso | Detalle |
|---|---|
| **Manejo diferenciado** | Los paquetes `Frágil` se almacenan solo en zonas Delicadas. Los `Peligrosos`, solo en zonas de Alto Riesgo. |
| **Trazabilidad completa** | Cada cambio de estado queda registrado con fecha/hora y usuario responsable. |
| **Notificación ante novedades** | Ante daño, extravío o devolución, se notifica al remitente (teléfono) y al destinatario (teléfono y correo) de forma automática. |

### Ante incidencias

| Situación | Compromiso |
|---|---|
| **Paquete dañado bajo custodia** | El `Módulo de Gestión de Finanzas` aplica descuento al transportador y/o cobro de póliza por el valor declarado. |
| **Paquete extraviado** | Se activa la indemnización hasta el valor declarado en la admisión. |
| **Paquete devuelto** | El remitente es notificado. El ajuste financiero por logística inversa se gestiona automáticamente. |
| **Corrección de tiempos estimados** | Si los datos físicos son corregidos en bodega, se re-emite la solicitud de ruta y se notifica al remitente con los nuevos tiempos. |

---

## 2. Lo que el servicio espera del cliente

### Información veraz y completa

| Obligación | Consecuencia si no se cumple |
|---|---|
| **Datos de remitente y destinatario correctos** | Las notificaciones no llegarán y las gestiones de novedad o devolución no podrán ejecutarse. |
| **Dirección de entrega precisa** | Una dirección incorrecta puede resultar en rechazo de ruta (`Excepción de Ruta`) o devolución con costo de logística inversa para el remitente. |
| **Declarar correctamente el tipo de contenido** | Si el contenido real difiere de lo declarado y causa daños, la responsabilidad recae en el remitente. El servicio no responde por mercancía peligrosa no declarada. |
| **Valor declarado veraz** | Un valor subdeclarado limita la indemnización. Un valor sobredeclarado puede derivar en ajustes y revisión del registro. |

### Empaque adecuado

| Obligación | Consecuencia si no se cumple |
|---|---|
| **Empaque apropiado para el tipo de mercancía** | Los daños causados por empaque insuficiente no están cubiertos por la póliza del servicio. Los paquetes `Frágil` deben llegar con protección interna adecuada (burbuja, espuma, relleno). |
| **Presentación física en sede para pesaje** | Sin pesaje presencial el registro no puede completarse. No se aceptan declaraciones de dimensiones o peso sin verificación. |

### Documentación para mercancía peligrosa

El remitente que declare mercancía `Peligrosa` debe presentar en el momento de la admisión la documentación de seguridad correspondiente al material (ficha técnica, hoja de seguridad u otro documento oficial que identifique el contenido y sus condiciones de manejo). Sin esta documentación el registro no puede completarse y el paquete no ingresa al sistema.

### Disponibilidad para comunicación

El teléfono del remitente debe estar activo para recibir notificaciones. Si un paquete devuelto queda en `En Espera de Instrucción` y el remitente no responde en 5 días hábiles, el Supervisor de Novedades define el siguiente paso de forma unilateral.

---

## 3. Limitaciones del servicio

| Limitación | Detalle |
|---|---|
| **Cobertura geográfica** | Solo se atienden destinos dentro de las zonas configuradas en el `Módulo de Gestión de Rutas`. Los destinos fuera de cobertura quedan en `Excepción de Ruta`. |
| **Límite de peso** | No se admiten paquetes con peso superior a 4.500 kg sin gestión manual especial. |
| **Responsabilidad limitada al valor declarado** | El monto máximo de indemnización es el valor declarado en la admisión. No se reconocen valores superiores. |
| **Tiempos de entrega referenciales** | Los tiempos estimados pueden variar por condiciones de tráfico, clima o disponibilidad de flota. |

---

## 4. Proceso de reclamaciones

| Situación | Cómo proceder |
|---|---|
| **Paquete dañado** | Reportar al Controlador de Novedades en sede con evidencia fotográfica. El sistema gestiona la compensación vía `Módulo de Gestión de Finanzas`. |
| **Paquete extraviado** | Contactar al Controlador de Novedades para iniciar el registro. La indemnización se activa por el valor declarado. |
| **Paquete devuelto** | El remitente recibe notificación automática y tiene 5 días hábiles para indicar instrucciones. |
| **Discrepancia en precio** | El precio es visible antes de confirmar el registro y queda fijo al confirmar. Las reclamaciones deben hacerse antes de confirmar. |
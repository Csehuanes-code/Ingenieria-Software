
# Sistema de Gestión Logística
## 📦 Módulo 1: Gestión de Paquetes

Este módulo debe garantizar la trazabilidad de los paquetes desde el minuto cero.

### Registro Detallado del Paquete (Atributos)
Para que el sistema sea robusto, cada paquete debe tener un **UUID (Identificador Único)** y los siguientes datos:
* **Logística:** ID único, Fecha/Hora de ingreso, Sede de origen.
* **Físicos:** Peso (kg/lb), Dimensiones (volumen), Tipo de mercancía (frágil, peligrosos, estándar).
* **Comercial:** Valor declarado (para seguros), Costo de envío, Método de pago (prepago o contra entrega).
* **Geográficos:** Dirección exacta con coordenadas GPS (lat/long) para facilitar el Módulo 2.

### Ciclo de Vida y Estados Expandidos
Proponemos un flujo de estados granular para evitar "puntos ciegos":
1. **Recibido en Sede:** El paquete entra al sistema.
2. **En Clasificación:** Se agrupa por zona de destino.
3. **Listo para Despacho:** Esperando asignación de ruta.
4. **En Tránsito:** Cargado en vehículo.
5. **En Parada de Entrega:** El transportador está en el sitio.
6. **Entregado:** Con firma y foto de evidencia (POD - Proof of Delivery).
7. **Novedad en Bodega:**
    * **Dañado:** Registro de avería con fotos.
    * **Extraviado:** Activación de protocolo de seguro.
    * **Devolución:** El paquete regresa al origen.

---

## 🚛 Módulo 2: Planificación de Rutas y Flota

Aquí es donde la eficiencia operativa ocurre. La clave es la **relación Peso-Capacidad**.

### Gestión de Inventario de Vehículos
Cada vehículo debe estar tipificado:
* **Tipos:** Moto (hasta 20kg), Van (hasta 500kg), Camión NHR (2 tons), Camión Turbo (4.5 tons).
* **Atributos:** Placa, Modelo, Conductor asignado, Volumen máximo (m³), **Peso Máximo Soportado**, zona el la que opera.

### Lógica de Asignación (Módulo 1 ↔ Módulo 2)
Cuando el Módulo 1 solicita una ruta, el Módulo 2 ejecuta un algoritmo de **Consolidación de Carga**:
1. **Filtrado por Zona:** Agrupa paquetes por cercanía geográfica.
2. **Selección de Vehículo:** El sistema busca el vehículo disponible más pequeño que soporte el peso total de los paquetes para optimizar combustible.
3. **Llenado por Capacidad:** Se van añadiendo paquetes hasta alcanzar el 90% de la capacidad de peso del vehículo (dejando un margen de seguridad).

### Gestión de Paradas y Novedades
Cada parada en la ruta debe registrar:
* **Estado de Parada:** Exitosa, Fallida, Reintento.
* **Motivos de No Entrega (Diccionario de Errores):**
    * Dirección incorrecta.
    * Cliente ausente.
    * Zona de difícil acceso / Orden público.
    * Rechazado por el cliente.

---

## 💰 Módulo 3: Facturación y Liquidación

Este módulo traduce la operación logística en datos financieros.

### Reglas de Negocio para el Pago
El sistema analizará el cierre de la ruta y aplicará la lógica según el **Tipo de Vehículo** (ya que un camión grande tiene costos operativos mayores que una moto).

**1. Modelos de Contratación:**
* **Recorrido Completo:** Se paga un valor fijo por la ruta (ideal para rutas rurales o largas distancias).
* **Por Parada Realizada:** Se paga por gestión (ideal para última milla urbana).

**2. Matriz de Pago por Estado de Entrega (Propuesta):**
Para que sea justo para el transportador y la empresa, el pago se pondera:

| Estado de la Entrega | % de Pago al Transportador | Justificación |
| :--- | :--- | :--- |
| **Entregado** | 100% | Servicio completado con éxito. |
| **Fallido (Culpa Cliente)** | 30% - 50% | Cubre el costo de desplazamiento y tiempo. |
| **Fallido (Culpa Transportador)** | 0% | Error de manipulación o logística del conductor. |
| **Dañado en Ruta** | 0% + Penalidad | El seguro del transportador debe responder. |




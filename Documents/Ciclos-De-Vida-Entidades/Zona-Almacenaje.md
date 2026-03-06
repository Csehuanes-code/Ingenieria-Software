# Zonas de Almacenaje — Módulo de Gestión de Paquetes (MOD1)

**Versión**: 1.0  
**Fecha**: 2026-02-28  
**Alcance**: Define las zonas de almacenamiento físico disponibles en bodega, sus atributos, las categorías de paquetes que admiten, los estados de zona, las reglas de operación y las pautas para los trabajadores que interactúan con ellas.

> **Distinción importante**: Las **Zonas de Almacenaje** descritas en este documento son espacios físicos en bodega donde se ubican temporalmente los paquetes durante su procesamiento. Son distintas de las **Zonas de Destino**, que son agrupaciones lógicas geográficas usadas para consolidar carga de vehículos (ver MOD1-UC-006). Un paquete tiene ambas asignadas al completar el flujo de almacenaje.

---

## 1. Atributos Comunes de Todas las Zonas de Almacenaje

Todas las zonas comparten el siguiente modelo de datos:

| Atributo | Tipo | Descripción |
|---|---|---|
| `id` | UUID (PK) | Identificador único de la zona |
| `nombre` | Texto | Nombre operativo de la zona (ej. "Zona Normal A-01") |
| `codigo` | Texto único | Código corto para identificación en etiquetas y pantallas (ej. "ZN-A01") |
| `categoria` | Enum | `Normal` \| `Delicada` \| `Alto Riesgo` \| `Retención` |
| `capacidad_max_kg` | Decimal | Peso máximo total acumulado de paquetes que puede contener |
| `capacidad_max_m3` | Decimal | Volumen máximo total acumulado de paquetes que puede contener |
| `capacidad_max_paquetes` | Entero | Número máximo de paquetes que puede contener simultáneamente |
| `peso_actual_kg` | Decimal | Suma del `peso_kg` de todos los paquetes actualmente asignados |
| `volumen_actual_m3` | Decimal | Suma del `volumen_m3` de todos los paquetes actualmente asignados |
| `contador_paquetes` | Entero | Número de paquetes actualmente asignados |
| `estado` | Enum | `Disponible` \| `Parcial` \| `Saturado` \| `Bloqueado` |
| `ubicacion_fisica` | Texto | Descripción de la ubicación en el plano de bodega (pasillo, nivel, posición) |
| `id_sede` | UUID (FK) | Sede a la que pertenece la zona |
| `zona_contingencia_id` | UUID (FK, nullable) | Zona alternativa sugerida cuando esta zona está saturada |

**Regla de actualización de contadores**: Toda asignación o remoción de un paquete a una zona debe actualizar los tres contadores (`peso_actual_kg`, `volumen_actual_m3`, `contador_paquetes`) de forma **atómica** en la misma operación de base de datos para mantener la consistencia.

---

## 2. Categorías de Zonas de Almacenaje

### 2.1 Zona Normal

**Descripción**: Espacio de almacenamiento estándar para paquetes que no requieren condiciones especiales de manejo o resguardo.

**Paquetes admitidos**:
- `tipo_mercancia = Estándar` únicamente.
- `categoria_carga = Normal` o `Carga Especial` (si el espacio es suficiente y hay equipo de carga disponible).
- Cualquier `prioridad` (`Estándar` o `Urgente`).

**Paquetes NO admitidos**:
- `tipo_mercancia = Frágil` → el sistema bloquea la asignación.
- `tipo_mercancia = Peligroso` → el sistema bloquea la asignación.

**Condiciones físicas**: Sin requisitos especiales de temperatura, humedad ni ventilación. Estantería estándar o racks industriales. Acceso libre a operarios de bodega.

**Capacidad típica configurada** (ajustable por Administrador del Sistema):
- `capacidad_max_kg`: 5.000 kg por zona.
- `capacidad_max_m3`: 20 m³ por zona.
- `capacidad_max_paquetes`: 200 paquetes por zona.

---

### 2.2 Zona Delicada

**Descripción**: Espacio de almacenamiento con condiciones especiales para mercancía susceptible de daño por impacto, vibración o presión.

**Paquetes admitidos**:
- `tipo_mercancia = Frágil` (obligatorio para estos paquetes).
- `tipo_mercancia = Estándar` también puede almacenarse aquí si la zona tiene capacidad disponible y el Almacenista lo decide.

**Paquetes NO admitidos**:
- `tipo_mercancia = Peligroso` → el sistema bloquea la asignación.

**Condiciones físicas**:
- Estantería con revestimiento acolchado o separadores de espuma.
- Señalización "Manejo Frágil" y "No apilar" visible en toda la zona.
- Acceso restringido: solo Almacenistas con capacitación en manejo de frágiles.
- Los paquetes deben colocarse en posición vertical con la indicación "Este lado arriba" respetada.
- Prohibido apilar paquetes frágiles sobre otros paquetes.

**Capacidad típica configurada**:
- `capacidad_max_kg`: 2.000 kg por zona.
- `capacidad_max_m3`: 8 m³ por zona.
- `capacidad_max_paquetes`: 100 paquetes por zona.

---

### 2.3 Zona Alto Riesgo

**Descripción**: Espacio de almacenamiento seguro y controlado para mercancía que representa riesgo de incendio, explosión, toxicidad, corrosión o reactividad.

**Paquetes admitidos**:
- `tipo_mercancia = Peligroso` (obligatorio para estos paquetes).

**Paquetes NO admitidos**:
- `tipo_mercancia = Estándar` ni `Frágil` → por eficiencia y seguridad, estas categorías no deben mezclarse con mercancía peligrosa.

**Condiciones físicas**:
- Área delimitada físicamente (paredes ignífugas o separación por distancia reglamentaria).
- Ventilación forzada y sistema de detección de gases / humo activo.
- Extintores específicos para el tipo de riesgo (polvo químico, CO₂) ubicados a no más de 5 metros de la entrada.
- Señalización de peligro, acceso restringido y uso obligatorio de EPP (guantes, gafas de protección, mascarilla según el tipo de material).
- Temperatura controlada si alguno de los materiales almacenados lo requiere (según MSDS).
- Registro físico en la puerta de la zona con todos los paquetes actualmente almacenados y su sustancia / material peligroso.

**Acceso**: Solo Almacenistas certificados en manejo de materiales peligrosos. El Supervisor de Bodega debe ser notificado antes de cualquier movimiento de paquetes en esta zona.

**Capacidad típica configurada**:
- `capacidad_max_kg`: 1.000 kg por zona.
- `capacidad_max_m3`: 4 m³ por zona.
- `capacidad_max_paquetes`: 30 paquetes por zona.

---

### 2.4 Zona de Retención

**Descripción**: Espacio de almacenamiento temporal para paquetes que no pueden continuar el flujo operativo normal por razones administrativas, legales o de inspección.

**Paquetes admitidos**:
- Cualquier `tipo_mercancia`, independientemente de la causa de retención.
- Paquetes en estado `Fuera de Tolerancia`, `Excepción de Ruta` o `En Espera de Instrucción`.
- Paquetes retenidos por orden de autoridad competente.
- Paquetes en análisis post-devolución.

**Condiciones físicas**: Área separada del flujo de paquetes activos. Acceso controlado por el Supervisor de Bodega. Cada paquete en esta zona debe tener un motivo de retención documentado en el sistema.

**Capacidad típica configurada**:
- `capacidad_max_kg`: 2.000 kg por zona.
- `capacidad_max_m3`: 8 m³ por zona.
- `capacidad_max_paquetes`: 50 paquetes por zona.

---

## 3. Estados de Zona

| Estado | Condición de activación | Impacto operativo |
|---|---|---|
| `Disponible` | Los tres contadores (`peso_actual_kg`, `volumen_actual_m3`, `contador_paquetes`) están por debajo del 80% de su capacidad máxima. | Flujo normal. El sistema sugiere esta zona para nuevas asignaciones. |
| `Parcial` | Al menos uno de los tres contadores superó el 80% pero ninguno alcanzó el 100% de su capacidad máxima. | El sistema puede seguir asignando paquetes, pero muestra una advertencia de capacidad próxima al límite al Almacenista. |
| `Saturado` | Al menos uno de los tres contadores alcanzó el 100% de su capacidad máxima. | El sistema bloquea nuevas asignaciones a esta zona, emite la alerta `Zona de Almacenamiento Saturada` y sugiere automáticamente la zona de contingencia configurada (`zona_contingencia_id`). |
| `Bloqueado` | El Supervisor de Bodega bloqueó manualmente la zona (por mantenimiento, incidente de seguridad, inspección, etc.). | El sistema impide cualquier nueva asignación o remoción de paquetes hasta que el Supervisor desbloquee la zona. Los paquetes actualmente en la zona quedan en espera. |

---

## 4. Flujo de Paquetes Respecto a las Zonas

### 4.1 Ingreso de un paquete a una zona

1. El Almacenista escanea el UUID del paquete en estado `Recibido en Sede`.
2. El sistema sugiere la zona más apropiada basándose en `tipo_mercancia` y coordenadas GPS del destinatario.
3. El Almacenista confirma la zona (puede seleccionar una diferente a la sugerida, dentro de las permitidas para el tipo de mercancía).
4. El sistema actualiza los contadores de la zona de forma atómica (`peso_actual_kg += peso_kg`, `volumen_actual_m3 += volumen_m3`, `contador_paquetes += 1`).
5. El estado del paquete cambia a `En Clasificación`.

### 4.2 Salida de un paquete de una zona

Un paquete sale de su zona de almacenamiento cuando:
- El Coordinador de Despacho lo escanea para iniciar la carga al vehículo (estado `Clasificado` → `En Carga`).
- El paquete es reasignado a otra zona por corrección de datos o excepción supervisada.
- El paquete entra a la Zona de Retención por decisión del Supervisor de Bodega.

Al salir, el sistema actualiza los contadores de la zona en sentido inverso (`peso_actual_kg -= peso_kg`, `volumen_actual_m3 -= volumen_m3`, `contador_paquetes -= 1`).

### 4.3 Reasignación de zona

Ocurre cuando:
- La zona original queda saturada y el paquete debe ir a la zona de contingencia.
- El Supervisor de Bodega decide mover un paquete por razones operativas.
- El cliente cambia la dirección de destino y la nueva clasificación por zona de destino implica un área de bodega diferente.

La reasignación actualiza los contadores de **ambas** zonas (origen y destino) de forma atómica. El paquete queda marcado con nota en el Historial de Estados indicando el motivo del cambio de zona.

---

## 5. Saturación de Zona — Procedimiento

### 5.1 Detección

El sistema detecta saturación automáticamente al intentar asignar un paquete y verifica que al menos uno de los tres contadores alcanzó su máximo.

### 5.2 Respuesta automática

1. El sistema emite la alerta `Zona de Almacenamiento Saturada` en la pantalla del Almacenista.
2. El sistema consulta la `zona_contingencia_id` de la zona saturada y verifica si tiene capacidad disponible.
3. Si la zona de contingencia tiene capacidad, el sistema sugiere esa zona como alternativa.
4. El Almacenista confirma la zona de contingencia y el paquete queda marcado con la nota `Desborde de zona — asignado a zona de contingencia`.

### 5.3 Cómo una zona pasa de Saturado a Disponible / Parcial

La zona actualiza su estado automáticamente cada vez que un paquete sale de ella (ya sea por carga al vehículo, reasignación o retención). El sistema recalcula los tres contadores tras cada salida y ajusta el estado:
- Si todos los contadores bajan del 100% → `Parcial`.
- Si todos los contadores bajan del 80% → `Disponible`.

No existe un proceso manual para "liberar" una zona. El estado es siempre un reflejo en tiempo real de los contadores.

### 5.4 Zona de contingencia no disponible

Si tanto la zona original como la zona de contingencia están saturadas, el sistema notifica al Supervisor de Bodega para que tome acción manual: habilitar una nueva zona temporal, reorganizar el inventario o pausar la admisión de nuevos paquetes para ese tipo de mercancía hasta liberar espacio.

---

## 6. Pautas para los Trabajadores

### 6.1 Almacenista

- **Verificar siempre la zona sugerida por el sistema** antes de confirmar. Si la zona sugerida no corresponde (ej. paquete frágil pero el sistema sugiere zona normal por error), reportar inmediatamente al Supervisor de Bodega.
- **No asignar manualmente paquetes a zonas no compatibles** con su categoría. El sistema bloquea esto, pero el Almacenista debe entender el motivo del bloqueo y no intentar saltarlo.
- **No colocar paquetes físicamente en una zona diferente a la asignada en el sistema**. La discrepancia entre la ubicación física real y la registrada en el sistema genera errores de trazabilidad.
- **Reportar inmediatamente al Supervisor de Bodega** si detecta que una zona alcanzó su capacidad física real antes de que el sistema la marque como saturada (ej. paquetes voluminosos que ocupan más espacio del calculado por volumen).
- **En la Zona Alto Riesgo**, siempre usar el EPP reglamentario y notificar al Supervisor antes de cualquier movimiento.

### 6.2 Coordinador de Despacho

- Al recoger un paquete de su zona para cargarlo al vehículo, **siempre escanear el UUID** para que el sistema actualice los contadores de la zona correctamente.
- **Verificar que el paquete recogido corresponde a la zona de destino y ruta asignada** antes de confirmar la carga al vehículo. Las discrepancias deben reportarse inmediatamente.
- No mover paquetes entre zonas sin escanear. El movimiento físico sin registro en el sistema genera inconsistencias en los contadores.

### 6.3 Supervisor de Bodega

- **Gestionar las zonas bloqueadas**: activar el bloqueo manual de zonas cuando sea necesario (mantenimiento, incidentes de seguridad, inspección) y desbloquearlas una vez resuelta la causa.
- **Resolver inconsistencias de tolerancia**: aprobar o rechazar los casos de `Fuera de Tolerancia — Peso` reportados por el Almacenista.
- **Autorizar excepciones**: aprobar la asignación excepcional a zonas no recomendadas cuando exista una justificación operativa válida y documentada.
- **Monitorear la Zona de Retención**: asegurarse de que todos los paquetes en retención tienen un motivo documentado y un tiempo estimado de resolución.

---

## 7. Relación con otros Documentos

| Documento | Relación |
|---|---|
| [MOD1-CICLO-DE-VIDA-PAQUETE.md](./MOD1-CICLO-DE-VIDA-PAQUETE.md) | Define los estados del paquete que determinan cuándo entra y sale de una zona. |
| [MOD1-CATEGORIAS-Y-ESTADOS-PAQUETE.md](./MOD1-CATEGORIAS-Y-ESTADOS-PAQUETE.md) | Define los tipos de mercancía que determinan qué zona de almacenaje debe usar cada paquete. |
| [MOD1-UC-004-Preparar-Paquete-Para-Almacenaje.md](./MOD1-UC-004-Preparar-Paquete-Para-Almacenaje.md) | Caso de uso que gestiona la asignación de zona de almacenaje. |
| [MOD1-UC-006-Clasificar-Paquete-Por-Zona-Destino.md](./MOD1-UC-006-Clasificar-Paquete-Por-Zona-Destino.md) | Caso de uso que gestiona la asignación de zona de destino lógica (diferente a la zona de almacenaje). |
| [MOD1-UC-007-Gestionar-Carga-De-Paquete.md](./MOD1-UC-007-Gestionar-Carga-De-Paquete.md) | Caso de uso que gestiona la salida del paquete de la zona hacia el vehículo. |
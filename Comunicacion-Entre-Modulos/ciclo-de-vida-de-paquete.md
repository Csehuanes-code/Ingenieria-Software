# Ciclo de Vida de un Paquete — Módulo de Gestión de Paquetes (MOD1)

**Versión**: 1.0  
**Fecha**: 2026-02-28  
**Alcance**: Describe todos los estados, transiciones, actores y ramificaciones posibles de un paquete desde su ingreso hasta el cierre financiero de su ciclo operativo.

---

## 1. Visión General

Un paquete nace en el momento en que el Empleado de Envío y Recepción inicia su registro y muere cuando el `Módulo de Gestión de Finanzas` confirma (ACK) el cierre contable de su UUID. Entre esos dos puntos, el paquete atraviesa una máquina de estados estricta: cada transición tiene un actor responsable, precondiciones obligatorias y una acción que la desencadena.

**Módulos externos que interactúan:**
- `Módulo de Gestión de Rutas` (M2): recibe la solicitud de ruta y emite el estado `En Tránsito` / `Entregado`.
- `Módulo de Gestión de Finanzas` (M3): recibe el estado final del paquete para ejecutar pagos o cobros.

---

## 2. Inventario de Estados

| Estado | Significado operativo | Actor que lo origina |
|---|---|---|
| `Borrador` | Registro iniciado pero no confirmado | Sistema (auto, al abrir formulario) |
| `Recibido en Sede` | Paquete admitido y pesado correctamente | Empleado de Envío y Recepción |
| `Pendiente GPS` | Dirección de destino sin coordenadas GPS válidas | Sistema (fallo geocodificador) |
| `Fuera de Tolerancia` | Inconsistencia bloqueante detectada en bodega | Sistema / Almacenista |
| `En Clasificación` | Zona de almacenamiento asignada; solicitud de ruta emitida a M2 | Almacenista |
| `Clasificado` | Zona de destino lógica asignada; listo para confirmar despacho | Almacenista |
| `Listo para Despacho` | Físicamente en andén de carga; habilitado para el Coordinador | Almacenista |
| `En Tránsito` | Cargado en vehículo y en ruta (emitido por M2) | Módulo de Gestión de Rutas |
| `Entregado` | Recibido por el destinatario (emitido por M2) | Módulo de Gestión de Rutas |
| `Dañado` | Avería física registrada con evidencia | Controlador de Novedades |
| `Extraviado` | No encontrado físicamente | Controlador de Novedades |
| `Devolución` | Retornado a sede por logística inversa | Controlador de Novedades |
| `Excepción de Ruta` | M2 rechazó la solicitud de ruta (cobertura o campo inválido) | Sistema |
| `Pendiente Sincronización Contable` | Novedad registrada; esperando ACK de M3 | Sistema |
| `Sincronizado Contablemente` | M3 confirmó recepción del estado final (cierre del ciclo) | Sistema |

---

## 3. Diagrama de Transiciones

```
[INICIO]
    │
    ▼
┌─────────────────────────────────────────────────────────────────────┐
│ ETAPA 1 — ADMISIÓN                                                  │
│ Actor: Empleado de Envío y Recepción                                │
│ Caso de Uso: MOD1-UC-001 + MOD1-UC-002                              │
└─────────────────────────────────────────────────────────────────────┘
    │
    ├──[Formulario abierto, UUID generado]──► Borrador
    │       │
    │       ├──[Inactivo > 30 min]──► ELIMINADO (job de limpieza FR-011)
    │       │
    │       └──[Empleado cancela]──► ELIMINADO (job de limpieza FR-011)
    │
    ├──[GPS falla durante registro]──► Recibido en Sede + gps_estado: Pendiente GPS
    │       │
    │       └──[Coordenadas resueltas manualmente]──► Recibido en Sede + gps_estado: Resuelto
    │                                                  (desbloquea avance a ETAPA 2)
    │
    └──[Pesaje completado + GPS resuelto + Confirmación]──► Recibido en Sede
                                                               │
                                                               ▼
                                                           [ETAPA 2]

┌─────────────────────────────────────────────────────────────────────┐
│ ETAPA 2 — ALMACENAJE EN BODEGA                                      │
│ Actor: Almacenista                                                  │
│ Caso de Uso: MOD1-UC-004 + MOD1-UC-006                              │
└─────────────────────────────────────────────────────────────────────┘
    │
    Recibido en Sede
    │
    ├──[GPS pendiente al escanear]──► Fuera de Tolerancia — Sin GPS
    │       │
    │       └──[Empleado resuelve coordenadas + Supervisor aprueba]──► Recibido en Sede
    │
    ├──[Diferencia de peso > 10% respecto a admisión]──► Fuera de Tolerancia — Peso
    │       │
    │       └──[Supervisor aprueba nuevo valor]──► Recibido en Sede
    │
    ├──[Zona de almacenamiento saturada]
    │       └──[Sistema sugiere zona de contingencia + Almacenista confirma]──► (continúa flujo normal)
    │
    └──[Zona de almacenamiento válida confirmada]
            │
            ├──[invocar MOD1-UC-006: Clasificar por Zona de Destino]
            │       ├──[Destino fuera de zonas configuradas]──► Supervisor asigna zona manualmente
            │       └──[Zona de destino asignada]──► estado: En Clasificación
            │
            └──[invocar MOD1-UC-003: Solicitar Ruta de Paquete → M2]
                    │
                    ├──[M2 rechaza: campo inválido o cobertura]──► Excepción de Ruta
                    │       │
                    │       └──[Supervisor corrige dato + Reintento]──► (re-ejecuta MOD1-UC-004)
                    │
                    ├──[M2 no responde]──► Evento encolado; estado permanece En Clasificación
                    │
                    └──[M2 responde ruta_asignada]──► id_ruta + id_transportador persistidos
                                                        estado: En Clasificación
                                                               │
                                                               ▼
                                                           [ETAPA 3]

┌─────────────────────────────────────────────────────────────────────┐
│ ETAPA 3 — CONFIRMACIÓN DE DISPONIBILIDAD                            │
│ Actor: Almacenista                                                  │
│ Caso de Uso: MOD1-UC-005                                            │
└─────────────────────────────────────────────────────────────────────┘
    │
    En Clasificación (con zona de destino asignada = Clasificado)
    │
    ├──[Daño detectado durante inspección visual]──► DERIVAR A ETAPA 6 (Novedad)
    │
    ├──[Fuera de Tolerancia pendiente]──► BLOQUEADO hasta resolución supervisada
    │
    └──[Sin inconsistencias + Confirmación]──► Listo para Despacho
                                                    │
                                                    ▼
                                                [ETAPA 4]

┌─────────────────────────────────────────────────────────────────────┐
│ ETAPA 4 — CARGA Y DESPACHO                                          │
│ Actor: Coordinador de Despacho                                      │
│ Caso de Uso: MOD1-UC-007                                            │
└─────────────────────────────────────────────────────────────────────┘
    │
    Listo para Despacho
    │
    ├──[id_transportador no disponible (M2 aún no respondió)]
    │       └──[BLOQUEADO hasta recibir ruta_asignada de M2]
    │
    ├──[Paquete marcado Dañado o Extraviado]──► BLOQUEADO; escalar a Controlador de Novedades
    │
    └──[Paquete válido + id_transportador disponible + Confirmación carga]
            │
            ├──[Se persiste Registro de Carga con id_transportador]
            │
            └──[Notificación a M2: carga completada]──► M2 emite estado En Tránsito
                                                               │
                                                               ▼
                                                           [ETAPA 5]

┌─────────────────────────────────────────────────────────────────────┐
│ ETAPA 5 — TRÁNSITO Y ENTREGA (Custodia del Módulo de Gestión de    │
│ Rutas)                                                              │
│ Actor externo: Módulo de Gestión de Rutas (M2)                      │
│ Caso de Uso: MOD1-UC-009 (al recibir estado final de M2)            │
└─────────────────────────────────────────────────────────────────────┘
    │
    En Tránsito (emitido por M2)
    │
    ├──[M2 reporta Entregado]──► invocar MOD1-UC-009
    │       └──► Pendiente Sincronización Contable
    │               └──[ACK de M3]──► Sincronizado Contablemente ──► [FIN DEL CICLO]
    │
    └──[M2 reporta novedad en ruta (Dañado / Extraviado / No entregado)]
            └──► DERIVAR A ETAPA 6 (Novedad)

┌─────────────────────────────────────────────────────────────────────┐
│ ETAPA 6 — GESTIÓN DE NOVEDADES (puede ocurrir en ETAPA 2, 3, 4 o 5)│
│ Actor: Controlador de Novedades                                     │
│ Caso de Uso: MOD1-UC-008 + MOD1-UC-009                             │
└─────────────────────────────────────────────────────────────────────┘
    │
    (Paquete en cualquier estado desde En Clasificación en adelante)
    │
    ├──[Tipo: Dañado]
    │       ├──[Con evidencia adjunta válida]
    │       │       ├──[invocar MOD1-UC-009]──► Pendiente Sincronización Contable
    │       │       │       └──[ACK de M3]──► Sincronizado Contablemente ──► [FIN]
    │       │       └──[M3 no responde]──► Reintentos automáticos c/ 5 min
    │       │
    │       └──[Sin evidencia]──► BLOQUEADO hasta adjuntar archivo válido
    │
    ├──[Tipo: Extraviado]
    │       ├──[invocar MOD1-UC-009]──► Pendiente Sincronización Contable
    │       │       └──[ACK de M3]──► Sincronizado Contablemente ──► [FIN]
    │       └──[M3 no responde]──► Reintentos automáticos c/ 5 min
    │
    └──[Tipo: Devolución]
            ├──[Re-ingreso a bodega]──► estado: Listo para Despacho
            │       └──► (Re-inicia ETAPA 4 para nuevo despacho)
            │
            └──[invocar MOD1-UC-009 (ajuste financiero por devolución)]
                    └──[ACK de M3]──► Sincronizado Contablemente
```

---

## 4. Descripción por Etapa

### Etapa 1 — Admisión

**Actor principal**: Empleado de Envío y Recepción  
**Casos de uso**: [MOD1-UC-001](./MOD1-UC-001-Registrar-Admision-De-Paquete.md), [MOD1-UC-002](./MOD1-UC-002-Procesar-Pesaje-Y-Dimensiones.md)

El paquete ingresa al sistema cuando el empleado abre el formulario de registro. En este punto se genera un UUID provisional y el registro queda en estado `Borrador`. El empleado captura los datos del remitente, destinatario, dirección de destino, valor declarado y método de pago. El sistema intenta resolver las coordenadas GPS de la dirección. Simultáneamente, se ejecuta el pesaje obligatorio (MOD1-UC-002), que captura `peso_kg`, dimensiones y tipo de mercancía.

**Casos de esta etapa:**

| Caso | Resultado |
|---|---|
| Todo completo y GPS resuelto | Estado `Recibido en Sede`; etiqueta ZPL emitida |
| GPS falla (timeout > 5s) | Estado `Recibido en Sede` con `gps_estado = Pendiente GPS`; bloqueado para clasificación |
| Empleado cancela o abandona formulario | Estado `Borrador`; eliminado automáticamente por job tras 30 min |
| Método de pago no soportado en la sede | Bloqueado; se muestra lista de métodos válidos |
| Pesaje no completado | Bloqueado; no se puede confirmar el registro |

---

### Etapa 2 — Almacenaje en Bodega

**Actor principal**: Almacenista  
**Casos de uso**: [MOD1-UC-004](./MOD1-UC-004-Preparar-Paquete-Para-Almacenaje.md), [MOD1-UC-006](./MOD1-UC-006-Clasificar-Paquete-Por-Zona-Destino.md), [MOD1-UC-003](./MOD1-UC-003-Solicitar-Ruta-De-Paquete.md)

El almacenista escanea el UUID. El sistema verifica las precondiciones y sugiere una zona de almacenamiento física basada en las coordenadas GPS y el tipo de mercancía. Al confirmar, se ejecutan en cadena: (1) clasificación por zona de destino lógica (MOD1-UC-006), (2) actualización de contadores de capacidad de la zona y (3) emisión **única** del evento `solicitar_ruta` al `Módulo de Gestión de Rutas` (MOD1-UC-003).

**Este es el único punto del ciclo de vida donde se emite `solicitar_ruta`.**

**Casos de esta etapa:**

| Caso | Resultado |
|---|---|
| Todo válido; M2 responde `ruta_asignada` | Estado `En Clasificación`; `id_ruta` e `id_transportador` persistidos |
| GPS pendiente al escanear | Estado `Fuera de Tolerancia — Sin GPS`; bloqueado |
| Diferencia de peso > 10% | Estado `Fuera de Tolerancia — Peso`; requiere aprobación supervisora |
| Zona de almacenamiento saturada | Sistema sugiere zona de contingencia; flujo continúa con zona alternativa |
| Destino sin zona configurada | Supervisor asigna zona de destino manualmente |
| M2 rechaza solicitud de ruta | Estado `Excepción de Ruta`; supervisor corrige dato y reintenta |
| M2 no responde | Evento encolado; estado `En Clasificación` se mantiene |
| Mercancía peligrosa en zona normal | Bloqueado; requiere zona de categoría `Alto Riesgo` |

---

### Etapa 3 — Confirmación de Disponibilidad

**Actor principal**: Almacenista  
**Caso de uso**: [MOD1-UC-005](./MOD1-UC-005-Actualizar-Estado-De-Disponibilidad.md)

Cuando el paquete completó la clasificación por zona de destino (estado `Clasificado`), el almacenista confirma que el embalaje está listo y el paquete está físicamente en el andén. Esta es una operación interna; no emite eventos externos.

**Casos de esta etapa:**

| Caso | Resultado |
|---|---|
| Estado `Clasificado`, sin inconsistencias | Estado `Listo para Despacho` |
| Estado diferente a `Clasificado` | Bloqueado; mensaje descriptivo de la transición requerida |
| `Fuera de Tolerancia` pendiente | Bloqueado hasta resolución supervisada |
| Daño detectado visualmente | No se permite la transición; derivar a Etapa 6 (Novedad) |

---

### Etapa 4 — Carga y Despacho

**Actor principal**: Coordinador de Despacho  
**Caso de uso**: [MOD1-UC-007](./MOD1-UC-007-Gestionar-Carga-De-Paquete.md)

El coordinador confirma la carga física del paquete al vehículo. El sistema crea el Registro de Carga con `id_transportador` (recuperado de la Solicitud de Ruta) y notifica al `Módulo de Gestión de Rutas`, que emite `En Tránsito`.

**Casos de esta etapa:**

| Caso | Resultado |
|---|---|
| `Listo para Despacho` + `id_transportador` disponible | Registro de Carga creado; M2 notificado; M2 emite `En Tránsito` |
| `id_transportador` no disponible (M2 aún no respondió) | Bloqueado hasta recibir `ruta_asignada` |
| Paquete marcado como `Dañado` o `Extraviado` | Bloqueado; escalar a Controlador de Novedades |
| M2 no responde a notificación de carga | Registro de Carga persiste; notificación encolada para reintento |
| Sesión del coordinador expira antes de confirmar | Sin operación registrada; paquete permanece `Listo para Despacho` |

---

### Etapa 5 — Tránsito y Entrega

**Actor externo**: Módulo de Gestión de Rutas (M2)  
**Caso de uso**: [MOD1-UC-009](./MOD1-UC-009-Informar-Estado-De-Paquete.md) (al recibir estado final)

Una vez en tránsito, la custodia está en manos de M2. El `Módulo de Gestión de Paquetes` actúa de forma reactiva: escucha los eventos de M2 y, al recibir un estado final, invoca MOD1-UC-009 para notificar a M3.

**Casos de esta etapa:**

| Caso | Resultado |
|---|---|
| M2 reporta `Entregado` | MOD1-UC-009 notifica a M3 → `Sincronizado Contablemente` |
| M2 reporta novedad (Dañado, Extraviado) | Derivar a Etapa 6 |
| M2 reporta no entregado (devuelto) | Derivar a Etapa 6 (tipo: `Devolución`) |

---

### Etapa 6 — Gestión de Novedades

**Actor principal**: Controlador de Novedades  
**Casos de uso**: [MOD1-UC-008](./MOD1-UC-008-Gestionar-Novedad-De-Paquete.md), [MOD1-UC-009](./MOD1-UC-009-Informar-Estado-De-Paquete.md)

Las novedades pueden ocurrir en cualquier momento desde `En Clasificación` en adelante. Cada novedad cierra con una notificación a M3.

**Casos por tipo de novedad:**

| Tipo | Requisito de evidencia | Estado posterior | Acción financiera (M3) |
|---|---|---|---|
| `Dañado` | Obligatoria (JPEG/PNG/MP4 ≤ 10 MB) | `Dañado` → `Pendiente Sincronización Contable` | Penalidad al transportador + cobro de póliza |
| `Extraviado` | Opcional | `Extraviado` → `Pendiente Sincronización Contable` | Indemnización por `valor_declarado` |
| `Devolución` | Opcional | `Listo para Despacho` (re-ingreso) | Pago parcial por logística inversa |
| `Devolución + Dañado` | Obligatoria (por el Dañado) | `Listo para Despacho` (re-ingreso) | Ambas acciones financieras |

---

## 5. Reglas de Integridad del Ciclo de Vida

1. **Unicidad del evento `solicitar_ruta`**: solo puede ser emitido desde MOD1-UC-004 al cambiar a `En Clasificación`. Ningún otro caso de uso puede dispararlo, salvo reintento supervisado por corrección de datos.

2. **Secuencia obligatoria de estados**: `Borrador` → `Recibido en Sede` → `En Clasificación` → `Clasificado` → `Listo para Despacho`. No se permiten saltos hacia adelante ni hacia atrás fuera de los escenarios de corrección definidos.

3. **`id_transportador` proviene exclusivamente del Registro de Carga**: el `id_transportador` que recibe M3 en MOD1-UC-009 debe ser el almacenado en el Registro de Carga, que a su vez lo recibió de la respuesta `ruta_asignada` de M2. No existe otro mecanismo de captura de este dato.

4. **Evidencia para novedades de tipo `Dañado` es irrenunciable**: sin archivo válido adjunto, el registro de novedad de tipo `Dañado` no puede guardarse ni puede invocarse MOD1-UC-009.

5. **Registros `Borrador` son efímeros**: el job de limpieza (FR-011 de MOD1-UC-001) elimina automáticamente los `Borrador` con más de 30 minutos de antigüedad, liberando el UUID para reutilización.

6. **El ciclo de vida cierra en `Sincronizado Contablemente`**: ningún paquete se considera procesado completamente hasta recibir el ACK de M3.

---

## 6. Trazabilidad de Casos de Uso por Estado

| Transición de Estado | Caso de Uso responsable |
|---|---|
| (ninguno) → `Borrador` | MOD1-UC-001 |
| `Borrador` → `Recibido en Sede` | MOD1-UC-001 + MOD1-UC-002 |
| `Recibido en Sede` → `En Clasificación` | MOD1-UC-004 + MOD1-UC-006 |
| `En Clasificación` → `Clasificado` | MOD1-UC-006 |
| `Clasificado` → `Listo para Despacho` | MOD1-UC-005 |
| Emitir evento `solicitar_ruta` a M2 | MOD1-UC-003 (invocado por MOD1-UC-004) |
| `Listo para Despacho` → `En Tránsito` | MOD1-UC-007 (notificación a M2) |
| `En Tránsito` → `Entregado` | M2 (externo) |
| Cualquier estado → `Dañado` / `Extraviado` / `Devolución` | MOD1-UC-008 |
| Cualquier estado final → `Pendiente Sincronización Contable` | MOD1-UC-009 |
| `Pendiente Sincronización Contable` → `Sincronizado Contablemente` | MOD1-UC-009 (ACK de M3) |
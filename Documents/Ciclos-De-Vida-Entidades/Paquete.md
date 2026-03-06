# Ciclo de Vida de un Paquete — Módulo de Gestión de Paquetes (MOD1)

**Versión**: 2.0  
**Fecha**: 2026-02-28  
**Alcance**: Describe todos los estados, transiciones, actores y ramificaciones posibles de un paquete desde su ingreso hasta el cierre financiero de su ciclo operativo.

---

## 1. Visión General

Un paquete nace cuando el Empleado de Envío y Recepción inicia su registro y cierra su ciclo cuando el `Módulo de Gestión de Finanzas` (M3) confirma el cierre contable de su UUID. Entre esos dos puntos, el paquete recorre una máquina de estados estricta: cada transición tiene un actor responsable, precondiciones obligatorias y una acción que la desencadena.

**Módulos externos que interactúan:**
- `Módulo de Gestión de Rutas` (M2): recibe la solicitud de ruta durante la admisión, asigna vehículo, ruta y tiempo estimado, y emite los estados `En Tránsito` y `Entregado`.
- `Módulo de Gestión de Finanzas` (M3): recibe el estado final del paquete para ejecutar pagos o cobros.

**Categorías que afectan el ciclo de vida**: Un paquete puede tener simultáneamente categorías de tipo de mercancía (`Estándar`, `Frágil`, `Peligroso`), categoría de carga (`Normal`, `Carga Especial`) y prioridad (`Estándar`, `Urgente`). Estas categorías determinan las zonas de almacenamiento válidas, los vehículos elegibles y los recargos tarifarios. Ver [Categorías y estados del paquete](./Categorias-Y-Estados-De-Paquete.md) para la definición completa de cada categoría.

---

## 2. Ciclo de Vida Ideal (Flujo Sin Incidencias)

```
Borrador → Recibido en Sede → En Clasificación → Clasificado → En Carga → Listo para Despacho → En Tránsito → Entregado → Pendiente Sincronización Contable → Sincronizado Contablemente
```

El evento [solicitar ruta](../Specs/MOD1-UC-003-Solicitar-Ruta-De-Paquete.md) al `Módulo de Gestión de Rutas` se emite **durante la admisión** (Etapa 1), inmediatamente tras el pesaje exitoso, cuando el paquete está en estado `Recibido en Sede` y tiene todos los datos requeridos por el Contrato de Integración.

---

## 3. Inventario de Estados

| Estado | Significado operativo | Actor que lo origina |
|---|---|---|
| `Borrador` | Registro iniciado pero no confirmado; pesaje pendiente | Sistema (auto, al abrir formulario) |
| `Recibido en Sede` | Paquete admitido, pesado y con solicitud de ruta emitida | Empleado de Envío y Recepción |
| `Pendiente GPS` | Dirección sin coordenadas GPS válidas; [solicitar ruta](../Specs/MOD1-UC-003-Solicitar-Ruta-De-Paquete.md) bloqueado | Sistema (fallo Google Maps Geocoding API) |
| `Fuera de Tolerancia` | Inconsistencia bloqueante en bodega (subtipo: `Peso` o `Sin GPS`) | Sistema / Almacenista |
| `En Clasificación` | Zona de almacenamiento física asignada en bodega | Almacenista |
| `Clasificado` | Zona de destino lógica asignada; paquete listo para ser cargado | Almacenista |
| `En Carga` | Coordinador recogió el paquete de bodega y lo está cargando al vehículo | Coordinador de Despacho |
| `Listo para Despacho` | Paquete físicamente ubicado y verificado en el vehículo | Coordinador de Despacho |
| `En Tránsito` | Vehículo despachado; paquete en ruta (emitido por M2) | Módulo de Gestión de Rutas |
| `Entregado` | Recibido por el destinatario con firma digital (POD) | Módulo de Gestión de Rutas |
| `Dañado` | Avería física registrada con evidencia | Controlador de Novedades |
| `Extraviado` | No encontrado físicamente | Controlador de Novedades |
| `Devolución` | Retornado a sede; en análisis post-devolución | Controlador de Novedades |
| `En Espera de Instrucción` | Análisis post-devolución requiere instrucción del remitente | Controlador de Novedades |
| `Excepción de Ruta` | M2 rechazó la solicitud de ruta (cobertura no disponible o campo inválido) | Sistema |
| `Pendiente Sincronización Contable` | Estado final registrado; esperando ACK de M3 | Sistema |
| `Sincronizado Contablemente` | M3 confirmó el cierre contable (fin del ciclo) | Sistema |

---

## 4. Diagrama de Transiciones

```
[INICIO]
    │
    ▼
┌─────────────────────────────────────────────────────────────────────┐
│ ETAPA 1 — ADMISIÓN                                                  │
│ Actor: Empleado de Envío y Recepción                                │
│ Casos de uso: MOD1-UC-001 + MOD1-UC-002 + MOD1-UC-003               │
└─────────────────────────────────────────────────────────────────────┘
    │
    ├──[Formulario abierto, UUID generado]──► Borrador
    │       ├──[Inactivo > 30 min]──► ELIMINADO (job de limpieza)
    │       └──[Empleado cancela]──► ELIMINADO
    │
    ├──[Google Maps API falla durante registro]
    │       └──[Pesaje completado + Confirmación]──► Recibido en Sede (gps_estado: Pendiente GPS)
    │               ├──[solicitar_ruta BLOQUEADO hasta GPS resuelto]
    │               └──[Coordenadas resueltas manualmente]──► gps_estado: Resuelto
    │                       └──[solicitar_ruta emitido a M2]──► (ver ramificaciones M2)
    │
    └──[Pesaje exitoso + GPS resuelto + Confirmación]──► Recibido en Sede
            └──[solicitar_ruta emitido automáticamente a M2]
                    ├──[M2 rechaza: cobertura o campo inválido]──► Excepción de Ruta
                    │       └──[Supervisor corrige dato + Reintento autorizado]──► (re-emite solicitar_ruta)
                    ├──[M2 no responde]──► Evento encolado; Recibido en Sede se mantiene
                    └──[M2 responde ruta_asignada]──► id_ruta + id_transportador + fecha_estimada persistidos
                                                        ──► fecha_estimada mostrada al cliente
                                                               │
                                                               ▼
                                                           [ETAPA 2]

┌─────────────────────────────────────────────────────────────────────┐
│ ETAPA 2 — ALMACENAJE EN BODEGA                                      │
│ Actor: Almacenista                                                  │
│ Casos de uso: MOD1-UC-004 + MOD1-UC-006                             │
└─────────────────────────────────────────────────────────────────────┘
    │
    Recibido en Sede
    │
    ├──[GPS pendiente al escanear]──► Fuera de Tolerancia — Sin GPS
    │       └──[Empleado resuelve coordenadas + Supervisor aprueba]──► Recibido en Sede
    │
    ├──[Diferencia de peso > 10%]──► Fuera de Tolerancia — Peso
    │       └──[Supervisor aprueba nuevo valor]──► Recibido en Sede
    │               └──[Si datos corregidos afectan payload]──► re-emite solicitar_ruta (autorizado)
    │
    ├──[Zona saturada]──► Sistema sugiere zona de contingencia
    │       └──[Almacenista confirma zona alternativa]──► (continúa flujo normal)
    │
    └──[Zona de almacenamiento asignada + MOD1-UC-006 ejecutado]──► En Clasificación
            │
            └──[Zona de destino asignada]──► Clasificado
                                                   │
                                                   ▼
                                               [ETAPA 3]

┌─────────────────────────────────────────────────────────────────────┐
│ ETAPA 3 — CONFIRMACIÓN DE DISPONIBILIDAD                            │
│ Actor: Almacenista                                                  │
│ Caso de uso: MOD1-UC-005                                            │
└─────────────────────────────────────────────────────────────────────┘
    │
    Clasificado
    │
    ├──[Daño detectado durante inspección visual]──► DERIVAR A ETAPA 6 (Novedad)
    ├──[Fuera de Tolerancia pendiente]──► BLOQUEADO hasta resolución supervisada
    │
    └──[Embalaje listo + Confirmación del almacenista]──► Listo para Despacho*
            │
            └──[Sistema notifica paquete_listo a M2]
                    ├──[M2 no responde]──► Notificación encolada; estado Listo para Despacho se mantiene
                    └──[M2 confirma]──► Paquete habilitado para carga
                                               │
                                               ▼
                                           [ETAPA 4]

* NOTA: El Coordinador de Despacho puede iniciar la carga directamente desde estado
  `Clasificado` (sin pasar por este paso si el paquete tiene ruta asignada).
  Ver Etapa 4.

┌─────────────────────────────────────────────────────────────────────┐
│ ETAPA 4 — CARGA Y DESPACHO                                          │
│ Actor: Coordinador de Despacho                                      │
│ Caso de uso: MOD1-UC-007                                            │
└─────────────────────────────────────────────────────────────────────┘
    │
    Clasificado (con id_ruta e id_transportador disponibles)
    │
    ├──[id_ruta o id_transportador no disponibles]──► BLOQUEADO; esperar respuesta M2
    ├──[Paquete Carga Especial sin equipo de carga disponible]──► BLOQUEADO
    ├──[Paquete marcado Dañado o Extraviado]──► BLOQUEADO; escalar a Controlador de Novedades
    │
    └──[Coordinador escanea UUID al recoger paquete]──► En Carga
            │
            ├──[Daño detectado durante manipulación]──► DERIVAR A ETAPA 6; revertir a Clasificado
            ├──[Discrepancia ruta/zona detectada]──► Alerta; escalar a Supervisor de Bodega
            │
            └──[Coordinador verifica vehículo + zona + ruta correctos; ubica en vehículo]
                    └──► Listo para Despacho
                            │
                            └──[Coordinador verifica todos los paquetes de la ruta en Listo para Despacho]
                                    └──[Confirma vehículo cargado]──► Notificación vehiculo_listo a M2
                                            ├──[M2 no responde]──► Notificación encolada
                                            └──[M2 confirma]──► M2 emite En Tránsito para cada paquete
                                                                       │
                                                                       ▼
                                                                   [ETAPA 5]

┌─────────────────────────────────────────────────────────────────────┐
│ ETAPA 5 — TRÁNSITO Y ENTREGA                                        │
│ Actor externo: Módulo de Gestión de Rutas (M2)                      │
│ Caso de uso: MOD1-UC-009 (al recibir estado final de M2)            │
└─────────────────────────────────────────────────────────────────────┘
    │
    En Tránsito
    │
    ├──[M2 reporta Entregado]──► invocar MOD1-UC-009
    │       └──► Pendiente Sincronización Contable
    │               └──[ACK de M3]──► Sincronizado Contablemente ──► [FIN DEL CICLO]
    │
    └──[M2 reporta novedad (Dañado / Extraviado / No entregado)]
            └──► DERIVAR A ETAPA 6 (Novedad)

┌─────────────────────────────────────────────────────────────────────┐
│ ETAPA 6 — GESTIÓN DE NOVEDADES                                      │
│ (puede ocurrir desde En Clasificación en adelante)                  │
│ Actor: Controlador de Novedades                                     │
│ Casos de uso: MOD1-UC-008 + MOD1-UC-009                             │
└─────────────────────────────────────────────────────────────────────┘
    │
    ├──[Tipo: Dañado]
    │       ├──[Sin evidencia]──► BLOQUEADO hasta adjuntar archivo válido
    │       └──[Con evidencia válida (JPEG/PNG/MP4 ≤ 10 MB)]
    │               ├──[Notificación a remitente y destinatario vía mensaje]
    │               └──[invocar MOD1-UC-009]──► Pendiente Sincronización Contable
    │                       └──[ACK de M3]──► Sincronizado Contablemente ──► [FIN]
    │
    ├──[Tipo: Extraviado]
    │       ├──[Notificación a remitente y destinatario vía mensaje]
    │       └──[invocar MOD1-UC-009]──► Pendiente Sincronización Contable
    │               └──[ACK de M3]──► Sincronizado Contablemente ──► [FIN]
    │
    └──[Tipo: Devolución]
            ├──[Notificación a remitente y destinatario vía mensaje]
            ├──[invocar MOD1-UC-009 — ajuste financiero por logística inversa]
            │
            └──[Análisis post-devolución obligatorio]
                    ├──[Paquete en buen estado]──► En Clasificación (por defecto)
                    │       └──► (Re-inicia Etapa 2 para nuevo despacho)
                    ├──[Daño detectado]──► Dañado ──► (ver rama Dañado)
                    ├──[Requiere instrucción del remitente]──► En Espera de Instrucción
                    │       ├──[Remitente responde: re-despacho]──► En Clasificación
                    │       ├──[Remitente recoge en sede]──► cierre
                    │       └──[Sin respuesta en 5 días hábiles]──► Supervisor define siguiente paso
                    └──[Contenido extraviado parcialmente]──► Extraviado ──► (ver rama Extraviado)
```

---

## 5. Descripción por Etapa

### Etapa 1 — Admisión

**Actor principal**: Empleado de Envío y Recepción  
**Casos de uso**: MOD1-UC-001, MOD1-UC-002, MOD1-UC-003

El paquete ingresa cuando el empleado abre el formulario de registro. Se genera un UUID provisional y el registro queda en estado `Borrador`. El empleado captura los datos del **Remitente** (`tipo_documento`, `numero_documento`, `nombre_completo`, `telefono`) y del **Destinatario** (`tipo_documento`, `numero_documento`, `nombre_completo`, `telefono`, `email`, `direccion_entrega`). El sistema resuelve las coordenadas GPS usando la **Google Maps Geocoding API** (timeout: 5 s).

Obligatoriamente se ejecuta MOD1-UC-002, que captura los **atributos físicos**: `peso_kg`, `largo_cm`, `ancho_cm`, `alto_cm`, `volumen_m3`, `peso_volumetrico_kg`, `tipo_mercancia` (`Estándar` | `Frágil` | `Peligroso`), `categoria_carga` (`Normal` | `Carga Especial`) y calcula el `precio_envio_calculado` con la fórmula de tarificación.

Una vez confirmado el registro y completado el pesaje exitosamente, si `gps_estado = Resuelto`, el sistema invoca MOD1-UC-003 e emite el evento `solicitar_ruta` al `Módulo de Gestión de Rutas`. La respuesta `ruta_asignada` incluye `id_ruta`, `id_transportador` y `fecha_estimada_entrega`, que se muestran al cliente y se persisten.

**Casos de esta etapa:**

| Caso | Resultado |
|---|---|
| Todo completo y GPS resuelto | `Recibido en Sede`; etiqueta ZPL emitida; `solicitar_ruta` emitido |
| GPS falla (timeout > 5s) | `Recibido en Sede` con `gps_estado = Pendiente GPS`; `solicitar_ruta` bloqueado |
| Empleado cancela o abandona | `Borrador`; eliminado por job tras 30 min |
| Método de pago no soportado | Bloqueado; se muestra lista de métodos válidos |
| Pesaje no completado | Bloqueado; no se puede confirmar el registro |
| M2 rechaza solicitud de ruta | `Excepción de Ruta`; Supervisor corrige y reintenta |
| M2 no responde | Evento encolado; `Recibido en Sede` se mantiene |

---

### Etapa 2 — Almacenaje en Bodega

**Actor principal**: Almacenista  
**Casos de uso**: MOD1-UC-004, MOD1-UC-006

El almacenista escanea el UUID. El sistema verifica precondiciones y sugiere la **zona de almacenamiento física** más apropiada según `tipo_mercancia` y coordenadas GPS. Al confirmar, se actualiza la zona (`peso_actual_kg`, `volumen_actual_m3`, `contador_paquetes`) y se invoca MOD1-UC-006 para asignar la zona de destino lógica. El paquete alcanza el estado `Clasificado`.

Ver [MOD1-ZONAS-DE-ALMACENAJE.md](./MOD1-ZONAS-DE-ALMACENAJE.md) para la descripción completa de cada zona.

---

### Etapa 3 — Confirmación de Disponibilidad

**Actor principal**: Almacenista  
**Caso de uso**: MOD1-UC-005

El almacenista confirma que el embalaje está listo y el paquete puede ir al andén. La transición permitida es `Clasificado` → `Listo para Despacho`. Al cambiar de estado, el sistema envía la notificación `paquete_listo` al `Módulo de Gestión de Rutas` (sin re-emitir `solicitar_ruta`).

---

### Etapa 4 — Carga y Despacho

**Actor principal**: Coordinador de Despacho  
**Caso de uso**: MOD1-UC-007

El coordinador opera sobre paquetes en estado `Clasificado` (con `id_ruta` e `id_transportador` disponibles). El flujo por paquete es: escaneo al recoger → estado `En Carga` → verificación en vehículo → estado `Listo para Despacho`. Una vez todos los paquetes de una ruta están en `Listo para Despacho`, el coordinador confirma el despacho del vehículo y el sistema emite `vehiculo_listo` a M2. M2 emite `En Tránsito` para cada paquete.

---

### Etapa 5 — Tránsito y Entrega

**Actor externo**: Módulo de Gestión de Rutas (M2)  
**Caso de uso**: MOD1-UC-009

El módulo de paquetes actúa reactivamente: escucha los eventos de M2 y, al recibir un estado final, invoca MOD1-UC-009 para notificar a M3.

---

### Etapa 6 — Gestión de Novedades

**Actor principal**: Controlador de Novedades  
**Casos de uso**: MOD1-UC-008, MOD1-UC-009

Puede ocurrir desde `En Clasificación` en adelante. El estado del paquete después de una devolución **no es `Listo para Despacho` por defecto**; es `En Clasificación` tras el análisis post-devolución, a menos que el análisis revele otra situación (daño, extravío, espera de instrucción).

**Casos por tipo de novedad:**

| Tipo | Evidencia | Notificación | Estado posterior al análisis | Acción financiera (M3) |
|---|---|---|---|---|
| `Dañado` | Obligatoria (JPEG/PNG/MP4 ≤ 10 MB) | Remitente + Destinatario | `Pendiente Sinc. Contable` | Penalidad + cobro póliza |
| `Extraviado` | Opcional | Remitente + Destinatario | `Pendiente Sinc. Contable` | Indemnización por `valor_declarado` |
| `Devolución` | Opcional | Remitente + Destinatario | `En Clasificación` (por defecto) | Pago logística inversa |
| `Devolución + Dañado` | Obligatoria | Remitente + Destinatario | `Dañado` → `Pendiente Sinc. Contable` | Ambas acciones |

---

## 6. Reglas de Integridad del Ciclo de Vida

1. **`solicitar_ruta` se emite durante la admisión**: se emite en Etapa 1 (estado `Recibido en Sede`) tras la combinación exitosa de MOD1-UC-001 + MOD1-UC-002. Solo puede re-emitirse en escenarios supervisados documentados (cobertura no disponible, corrección de datos físicos, cambio de dirección autorizado).

2. **Secuencia obligatoria del ciclo ideal**: `Borrador` → `Recibido en Sede` → `En Clasificación` → `Clasificado` → `En Carga` → `Listo para Despacho`. No se permiten saltos fuera de los escenarios de corrección definidos.

3. **`id_transportador` proviene exclusivamente del Registro de Carga**: creado en MOD1-UC-007, que lo toma de la entidad Solicitud de Ruta, que a su vez lo recibió de M2. No existe otra fuente válida.

4. **Evidencia para novedades tipo `Dañado` es irrenunciable**: sin archivo válido adjunto, el registro no puede guardarse.

5. **`Borrador` es efímero**: eliminado automáticamente tras 30 min de inactividad.

6. **El ciclo cierra en `Sincronizado Contablemente`**: ningún paquete se considera procesado completamente hasta el ACK de M3.

7. **Las devoluciones no implican estado `Listo para Despacho` automático**: el estado post-devolución depende del análisis físico del paquete. El estado por defecto es `En Clasificación`.

8. **El estado `Listo para Despacho` es irreversible sin autorización del Supervisor de Bodega**: ningún actor operativo puede revertir unilateralmente un paquete que ya está en `Listo para Despacho`.

---

## 7. Trazabilidad de Casos de Uso por Estado

| Transición | Caso de uso responsable |
|---|---|
| (ninguno) → `Borrador` | MOD1-UC-001 |
| `Borrador` → `Recibido en Sede` | MOD1-UC-001 + MOD1-UC-002 |
| Emitir `solicitar_ruta` a M2 | MOD1-UC-003 (invocado por MOD1-UC-001 tras pesaje exitoso) |
| `Recibido en Sede` → `En Clasificación` | MOD1-UC-004 |
| `En Clasificación` → `Clasificado` | MOD1-UC-006 |
| `Clasificado` → `Listo para Despacho` | MOD1-UC-005 (confirmación de embalaje) |
| `Clasificado` → `En Carga` | MOD1-UC-007 (escaneo por Coordinador) |
| `En Carga` → `Listo para Despacho` | MOD1-UC-007 (verificación en vehículo) |
| `Listo para Despacho` → `En Tránsito` | MOD1-UC-007 (notificación `vehiculo_listo`) + M2 |
| `En Tránsito` → `Entregado` | M2 (externo) |
| Cualquier estado → `Dañado` / `Extraviado` / `Devolución` | MOD1-UC-008 |
| Estado final → `Pendiente Sincronización Contable` | MOD1-UC-009 |
| `Pendiente Sinc. Contable` → `Sincronizado Contablemente` | MOD1-UC-009 (ACK de M3) |
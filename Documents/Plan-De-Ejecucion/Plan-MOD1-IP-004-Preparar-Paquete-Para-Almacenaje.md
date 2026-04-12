# Implementation Plan: Preparar Paquete para Almacenaje (MOD1-IP-004)

**Date:** 2026-04-11
**Spec:** [Preparar Paquete para Almacenaje](../Specs/MOD1-UC-004-Preparar-Paquete-Para-Almacenaje.md)
**Arquitectura:** Ver [Metodologias](../Metodologias.md)
**Orden de ejecución:** Sprint 2 — Sujeto a la disponibilidad del paquete en bodega.

---

## Summary

Permite al Almacenista asignar físicamente un paquete admitido a una zona de almacenamiento. Valida la categoría de mercancías (ej. bloqueando ítems peligrosos en áreas normales) e impacta atómicamente la ocupación volumétrica y de pesaje en tiempo real en la Base de Datos. Implementa un sistema en React UI alimentado por Backend Java 21, capaz de canalizar el flujo hacia gestión de novedades (MOD1-IP-006) en caso de discrepancias métricas o de daños.

---

## Technical Context

| Campo | Valor |
|---|---|
| **Language/Version** | Java 21 (Backend) / JavaScript (React para Frontend) |
| **Primary Dependencies** | Spring Web, Spring Data JPA, Hibernate, PostgreSQL, Gradle |
| **Storage** | PostgreSQL (Control Concurrente y Bloqueo Optimista para el contador de inventario de las Zonas) |
| **Testing** | JUnit 5, Mockito, Testcontainers (PostgreSQL) para concurrencia |
| **Target Platform** | Servidor Linux para Interfaz de Datos API / PDA o Terminal Frontend React para Almacenista |
| **Project Type** | Web application (Sistemas internos operativos) |
| **Performance Goals** | Evitar Race Conditions extremas al actualizar Zonas Centrales concurrentemente |
| **Constraints** | Bloqueo algorítmico estricto en zonas saturadas. El límite de capacidades obliga a zonas de contingencia. |
| **Scale/Scope** | Alto impacto transversal intrainventarios. Riesgo de interbloqueos de bases de datos. |
| **Framework** | Spring Boot 3.x (Backend), React (Frontend) |
| **Arquitectura** | Hexagonal (Ports & Adapters) — Backend aislado |

---

## Project Structure

> La manipulación y reglas en capas modulares exigen controles concurrentes transaccionales de control optimista/pesimista sobre las entidades en repositorios aislados de la lógica.

```text
frontend/
└── src/
    ├── components/
    │   └── forms/
    │       └── ZonaAsignacionForm.jsx         [NUEVO — Scanner UI para UUIDs]
    └── services/
        └── BodegaApiService.js                [NUEVO]

backend/
├── domain/
│   ├── model/
│   │   ├── Paquete.java                       [MODIFICADO — De Fase 2 IP-001 y 002]
│   │   ├── ZonaAlmacenamiento.java            [NUEVO — Maneja contadores volumétricos propios]
│   │   └── DiscrepanciaFisica.java            [NUEVO — Tipo de dato rastreable en historiales]
│   ├── exception/
│   │   ├── ZonaSaturadaException.java         [NUEVO]
│   │   └── MercanciaIncompatibleException.java[NUEVO]
│   └── external/
│           ├── PaqueteRepository.java         [REUTILIZADO]
│           └── ZonaRepository.java            [NUEVO]
│
├── application/
│   └── bodega/
│       └── AsignarZonaBodegaUseCase.java      [NUEVO — Orquestador concurrencia y validaciones]
│
└── infrastructure/
    ├── adapter/
    │   ├── in/
    │   │   └── web/
    │   │       └── BodegaController.java      [NUEVO]
    │   └── out/
    │       └── persistence/
    │           ├── ZonaJpaAdapter.java        [NUEVO]
    │           └── ZonaRepository.java        [NUEVO — Usa @Version / Optimistic Lock]
    └── dto/
        └── request/
            └── AsignacionZonaRequest.java     [NUEVO]
```

---

## Phase 1: Prerequisitos 

- [ ] T101 Validar creación estructural y tablas base inicializadas (`zona_almacenamientos`, `paquetes`) en los scripts de Flyways respetando soporte concurrente (agregando columnas `@Version` necesarias según `JPA OptimisticLocking`).

---

## Phase 2: Dominio de Zonas Físicas y Control de Capacidad

**Purpose:** Validación rígida, asegurando matemáticamente en memoria que las tolerancias pre-programadas de volúmenes nunca se excedan y la naturaleza se respete.

### Tests del algoritmo (TDD)

- [ ] T102 [P] Test unitario `ZonaAlmacenamientoTest` Capacidad:
  - Validar inyección volumétrica: un objeto Zona rechaza e invoca `ZonaSaturadaException` si al sumar el volumen del Paquete analizado las capacidades predeterminadas se vencen.
- [ ] T103 [P] Test unitario `ZonaAlmacenamientoTest` Naturaleza:
  - Incompatibilidad: Validar rechazo directo de instanciar o relacionarse sobre mercancía `Peligroso` contra Zonas Normales sin flag.

### Implementación del dominio

- [ ] T104 [P] Modelar e inhibir saturación estática del objeto `ZonaAlmacenamiento`:
```java
// backend/domain/model/ZonaAlmacenamiento.java
public class ZonaAlmacenamiento {
    private UUID id;
    private Long version; // Control Concurrente
    private CategoriaZona categoria;
    
    // Contadores actuales
    private Double kilosAcumulados;
    private Double capacidadKilos;
    private Integer paquetesAlmacenados;

    public void registrarArriboPaquete(Paquete pq) {
        if (!pq.getTipoMercancia().estaPermitidoEn(this.categoria)) {
            throw new MercanciaIncompatibleException("Prohibido almacenar mercancia especial en zona estándar.");
        }
        if ((kilosAcumulados + pq.getPeso().getValor()) > capacidadKilos) {
            throw new ZonaSaturadaException(); 
        }
        
        this.kilosAcumulados += pq.getPeso().getValor();
        this.paquetesAlmacenados++;
    }
}
```

---

## Phase 3: Servicio de Aplicación — AsignarZonaBodegaUseCase (US4)

**Goal:** Recepciona directivas de almacenistas, efectúa la reserva concurrente del espacio, loguea discrepancias en caso de correcciones, y efectúa cambios atómicos de estado.

### Tests del servicio (TDD)

- [ ] T105 [P] [US4] Asignación Normal e intercepción lógica:
  - Previene que dos almacenistas envíen a ruta un UUID al mismo tiempo disparando alerta semántica interceptando al segundo hilo con el `ObjectOptimisticLockingFailureException` del repositorio sobre el port asumiendo su abstracción.
- [ ] T106 [P] [US4] Asignación de Novedad por Extravío:
  - Simula y avala invocaciones cruzadas (inyección por Módulo dependiente 006) en caso de Paquete no hallado localizable.

### Implementación del servicio

- [ ] T107 [P] [US4] Desarrollar motor relacional centralizado:
```java
// backend/application/bodega/AsignarZonaBodegaUseCase.java
@Service
@Transactional
public class AsignarZonaBodegaUseCase {

    private final ZonaRepository zonas;
    private final PaqueteRepository paquetes;

    @Override
    public UUID acomodarPaquete(UUID paqueteId, UUID zonaIdSugerida) {
        Paquete paquete = paquetes.buscarParaUpdate(paqueteId); // Lock pesimista a nivel SQL
        ZonaAlmacenamiento zona = zonas.buscarGarantizandoNoSaturacion(zonaIdSugerida); // Lock optimista
        
        zona.registrarArriboPaquete(paquete);
        paquete.aplicarZonaFisica(zonaIdSugerida);

        zonas.guardar(zona); // Puede rebotar por race conditions aquí. 
        paquetes.guardar(paquete);
        
        // El controller o un Event Listener capturará y redirigirá la ejecución obligatoria 
        // a [MOD1-UC-005] como parte de un SAGA si aplicase.
        return paqueteId;
    }
}
```

---

## Phase 4: Adaptadores de Interfaz e Interconectividad UI

### Adaptador REST (Backend)

- [ ] T108 [US4] Endpoints POST en `BodegaController.java`. Requiere Exception Handlers especializados para capturar las `ZonaSaturadaException` informando en su JSON Body un fallback hacia una lista de Zonas de Contingencia viables.

### Aplicación UI (Frontend Javascript/React)

- [ ] T109 [US4] App para escáner en bodegas `ZonaAsignacionForm.jsx`: Desplegar dinámicamente un Dropdown List responsivo que sólo presente visualmente Zonas habilitadas a la tipología detectada del Paquete recién escaneado, ahorrando frustración del usuario.
- [ ] T110 [US4] Componente de Discrepancias Visual: Un botón toggleable en UI para activar flujos secundarios que invocarán métodos en Backend orientados a corrección silogística antes de asentarse.

---

## Phase N: Polish

- [ ] T111 Test Integral concurrente (JUnit + `@DataJpaTest` o similar en Testcontainers usando multi-threading) induciendo 5 hilos simulados enviando `acomodarPaquete(uuid)` para asegurar un límite exacto en la ocupación volumétrica y un contador impecable.

---

## Dependencies & Execution Order

```text
SPRINT BASE
    └── MOD1-IP-001 / MOD1-IP-002
            └── Este Plan ---> Phase 2, 3, 4
```

---

## Notes

- **Precaución concurrente:** Asegurar el uso de locks optimistas en la base de datos de Zonas.

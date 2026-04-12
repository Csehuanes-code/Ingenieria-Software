# Implementation Plan: Clasificar Paquete por Zona de Destino (MOD1-IP-005)

**Date:** 2026-04-11
**Spec:** [Clasificar Paquete por Zona de Destino](../Specs/MOD1-UC-005-Clasificar-Paquete-Por-Zona-Destino.md)
**Arquitectura:** Ver [Metodologias](../Metodologias.md)
**Orden de ejecución:** Sprint 2 — Automático post-almacenaje físico.

---

## Summary

La clasificación lógica por "Zona Destino" agrupa los paquetes basados en la proximidad geográfica final del cliente (ej. Norte, Occidente), para habilitar algoritmos de consolidación y eficiencias logísticas. Desarrollado nativamente sobre las directrices TDD y Clean Architecture de Java 21, requiere validación cruzada automática en el instante que la asignación física culmina, forzando transiciones inmutables del estado al ciclo de `Listo Para Despacho` tras confirmación UI.

---

## Technical Context

| Campo | Valor |
|---|---|
| **Language/Version** | Java 21 (Backend) / JavaScript (React para Frontend) |
| **Primary Dependencies** | Spring Web, Spring Data JPA, PostgreSQL, Gradle |
| **Storage** | PostgreSQL (Indices geo-espaciales o referencias indexadas a regiones) |
| **Testing** | JUnit 5, Mockito |
| **Target Platform** | Backend API Rest interna / React SPA en inventarios |
| **Project Type** | Web application (Integración en backoffice geográfico) |
| **Performance Goals** | Cálculos de intersección lógicas en < 2 segundos por paquete aislado. |
| **Constraints** | Obligatoreidad de mantener restricciones cruzadas (Fragilidad no aplicable contra áreas de destino saturadas globalmente). |
| **Scale/Scope** | Lógico. Permite la orquestación masiva y escalabilidad a herramientas de ML futuras. |
| **Framework** | Spring Boot 3.x (Backend), React (Frontend) |
| **Arquitectura** | Hexagonal (Ports & Adapters) — Dominio Analítico |

---

## Project Structure

> Aislamiento estricto. Separando geofences o lógicas espaciales de la tecnología.

```text
backend/
├── domain/
│   ├── model/
│   │   ├── Paquete.java                       [MANTIENE LA ESTRUCTURA - Update zonaDestinoId]
│   │   └── ZonaDestinoGeografica.java         [NUEVO — Entidad abstracta representativa de región]
│   ├── exception/
│   │   └── IncongruenciaRegionalException.java[NUEVO]
│   └── external/
│           ├── MapasRegionales.java           [NUEVO — Consulta al sistema de regiones]
│           └── LogTransicionRepository.java   [NUEVO - Acatando FR-002: Registro persistente]
│
├── application/
│   └── geoclasificacion/
│       └── ClasificadorDestinosUseCase.java   [NUEVO — Orquestador de clasificador]
│
└── infrastructure/
    ├── adapter/
    │   ├── in/
    │   │   └── web/
    │   │       └── DestinosController.java    [NUEVO]
    │   └── out/
    │       └── persistence/
    │           ├── ZonaDestinoJpaAdapter.java [NUEVO]
    └── dto/
        └── request/
            └── ConfirmacionClasificacionRequest.java [NUEVO]
```

---

## Phase 1: Prerequisitos 

- [ ] T101 Migración de las tablas maestras iniciales correspondientes a las `Zonas Destino` (las cuales deben diferir completamente de las tablas de Zonas de Almacenamiento físicas construidas en el IP-004), alimentándolas con seeds funcionales (Regiones locales de pruebas).

---

## Phase 2: Dominio Lógico - Asignaciones Geográficas Evaluadas

**Purpose:** Validar, asilar y asegurar matemáticamente la concordancia de las etiquetas geonaturales al perfil de la instancia `Paquete`. 

### Tests del algoritmo (TDD puro)

- [ ] T102 [P] Test unitario `ZonaDestinoTest`:
  - Bloqueos condicionales restrictivos: Una Zona Destino marcada temporalmente por saturación de despachos arrojará rebote o advertencias en cascada en la ejecución central.
  - Rechazar vinculaciones inmutables directas para bienes "Frágiles" en polígonos lógicos no equipados administrativamente para ellos (FR-003).

### Implementación del dominio

- [ ] T103 [P] Construcción del clasificador y validador semántico:
```java
// backend/domain/model/ZonaDestinoGeografica.java
public class ZonaDestinoGeografica {
    private String codigoRegional;
    private Boolean admiteCategoriasPeligrosas;
    private Integer limiteOperativoTemporal;
    // ...

    public void analizarCompatibilidad(Paquete paquete) {
        if(paquete.getTipoMercancia() == TipoMercancia.PELIGROSO && !admiteCategoriasPeligrosas){
            throw new IncongruenciaRegionalException("La sub-zona logística carece de unidades acorazadas locales válidas.");
        }
        // Más reglas locales
    }
}
```

---

## Phase 3: Servicio de Aplicación — ClasificadorDestinosUseCase (US5)

**Goal:** Centralizar y mutar el estado del paquete globalmente desde `En Clasificación` al ciclo definitivo pre-asignaciones a camiones `Listo para Despacho`.

### Tests del servicio (TDD)

- [ ] T104 [P] [US5] Aplicación formal transaccional y flujo exitoso:
  - Verificar en JUnit Mockeado la integración contra Mock Puertos Georeferenciales, y afirmar que el paquete al concluir invocará a las funciones mutables y asoma un `paquetes.guardar(paquete_listo)`.

### Implementación del servicio

- [ ] T105 [P] [US5] Implementar `ClasificadorDestinosUseCase` controlando el flujo determinista y emitiendo los comprobantes (FR-002).
```java
// backend/application/geoclasificacion/ClasificadorDestinosUseCase.java
@Service
@Transactional
public class ClasificadorDestinosUseCase {
    private final PaqueteRepository paquetes;
    private final MapasRegionales poligonos;
    // ...

    @Override
    public ConfirmacionClasificacion ejecutarClasificacionLogica(UUID paqueteId) {
        Paquete paquete = paquetes.buscarPorId(paqueteId); 
        
        ZonaDestinoGeografica areaAplicable = poligonos.buscarRegionCandidataOptima(paquete.getCoordenadas());
        
        areaAplicable.analizarCompatibilidad(paquete); // Chequea bloqueos y fragilidades locales
        
        paquete.aplicarZonaRegionalDestinacion(areaAplicable.getId()); // Mutación Estado
        
        paquetes.guardar(paquete);
        
        // Emisión y creación del comprobante de trazabilidad exigido (FR-002)
        return generarYGuardarRegistroClasificacion(areaAplicable, paquete);
    }
}
```

---

## Phase 4: Interfaces y Exuberancia REST

### Adaptador REST (Backend)

- [ ] T106 [US5] Disponer un endpoint GET que devuelva las alternativas sugeridas interactivas para confirmación humana (Frontend) en lugar de imposiciones ciegas (alineándose a la confirmación de UI requerida en la Spec), y un POST de afirmación y ejecución transaccional definitiva.

### Interfaz UI React Frontend

- [ ] T107 [US5] Modales condicionales y previsualizadores textuales de la región (Ej. `Asignar a Zona Norte Automáticamente - Procediendo`). Despliegue en la aplicación PDA del almacenista que acaba de re-alocar físicamente (UX continuo encadenando IP-004 e IP-005 en una simple wizard-view).

---

## Phase N: Polish

- [ ] T108 Ajuste fino visual (React Spinner o Loading Skeletons en Tiempos de carga) asumiendo que los motores espaciales podrían presentar lentitud temporal si su infraestructura escala ineficientemente en PostGIS (Testing Integral).

---

## Dependencies & Execution Order

```text
SPRINT BASE
    └── MOD1-IP-001 / MOD1-IP-004
            └── Este Plan ---> Phase 2, 3, 4
```

---

## Notes

- **Servicio geo-espacial externo:** Cuidado con falsos positivos en zonas limítrofes. Asegurar control visual en frontend.

# Implementation Plan: Procesar Pesaje y Dimensiones (MOD1-IP-002)

**Date:** 2026-04-11
**Spec:** [Procesar Pesaje Y Dimensiones](../Specs/MOD1-UC-002-Procesar-Pesaje-Y-Dimensiones.md)
**Arquitectura:** Ver [Metodologias](../Metodologias.md)
**Orden de ejecución:** Sprint 1 — Inmediatamente posterior o en paralelo a Admisión (MOD1-IP-001)

---

## Summary

Como Empleado de Envío y Recepción, necesito capturar el peso, dimensiones y el tipo de mercancía de un paquete para generar el precio del envío mediante fórmulas predeterminadas. La implementación usará React en Frontend para las alertas predictivas (SPA) y Spring Boot (Java 21) en core Hexagonal aislando el poderoso y riguroso algoritmo tarifario, permitiendo persistir atómicamente el paquete amplificado en PostgreSQL e invocando externamente los avisos requeridos.

---

## Technical Context

| Campo | Valor |
|---|---|
| **Language/Version** | Java 21 (Backend) / JavaScript (React para Frontend) |
| **Primary Dependencies** | Spring Web, Spring Data JPA, Spring Validation, PostgreSQL, Gradle |
| **Storage** | PostgreSQL (Guardado atómico inmutable para tarifas) |
| **Testing** | JUnit 5, Mockito, Testcontainers (PostgreSQL) |
| **Target Platform** | Servidor Linux para Backend API Rest / Web Browser para UI |
| **Project Type** | Single Web Application (Aplicación web centralizada) |
| **Performance Goals** | Cálculos algorítmicos ultra-rápidos en < 10s |
| **Constraints** | Restricción algorítmica: medidas > 0 en su totalidad; Límite duro bloqueante de 70kg, Arquitectura Hexagonal. |
| **Scale/Scope** | Módulo de cálculo intensivo y bloqueante, crítico en la generación de cobros fijos en las sedes. |
| **Framework** | Spring Boot 3.x (Backend), React (Frontend) |
| **Arquitectura** | Hexagonal (Ports & Adapters) — Backend aislado |

---

## Project Structure

> Aplicación de la Estructura Hexagonal según guía de agentes.

```text
frontend/
└── src/
    ├── components/
    │   └── form/
    │       └── PesajeDimensionesForm.jsx      [NUEVO — Componente React para recolección física]
    └── services/
        └── PesajeApiService.js                [NUEVO]

backend/
├── domain/
│   ├── model/
│   │   ├── Paquete.java                       [MODIFICADO — De Fase 2 IP-001]
│   │   ├── Peso.java                          [NUEVO — Value Object validando min/max absolutos]
│   │   ├── Dimensiones.java                   [NUEVO — Value Object l,w,h y cálculo de áreas m3]
│   │   └── DesgloseTarifario.java             [NUEVO — Lógica del FR-007 inyectada libre del framework]
│   ├── exception/
│   │   ├── SuperaLimiteCargaException.java    [NUEVO]
│   │   └── DimensionInvalidaException.java    [NUEVO]
│   └── external/
│           ├── PaqueteRepository.java         [REUTILIZADO — IP-001]
│           └── ConfiguracionTarifas.java      [NUEVO — Fetch a diccionario global externo de red]
│
├── application/
│   └── pesaje/
│       └── ProcesarPesajeUseCase.java         [NUEVO — Orquestador de lógica de negocio]
│
└── infrastructure/
    ├── adapter/
    │   ├── in/
    │   │   └── web/
    │   │       └── PesajeController.java      [NUEVO — Receptor HTTP]
    │   └── out/
    │       └── persistence/
    │           └── ConfiguracionTarifaJpa.java[NUEVO]
    └── dto/
        └── request/
            └── RegistroPesajeRequest.java     [NUEVO]
```

---

## Phase 1: Prerequisitos (verificación)

> **Dependencia bloqueante:** Llenado y persistencia parcial de MOD1-IP-001 (Setup Inicial y Tabla en Base de Datos).

- [ ] T101 Validar que la Entidad de Dominio Consolidada `Paquete` ya cuenta con los atributos (`peso`, `dimensiones`, `precioEnvio`) de las especificidades conjuntas.
- [ ] T102 Construir script Flyway `VX__add_fisics_to_paquetes.sql` o `V1` según inicialización, para columnas del tipo `NUMERIC(10,2)` en métricas y precios.

---

## Phase 2: Dominio Físico y Tarifario

**Purpose:** Implementación pura de algoritmos tarifarios sin inyección de dependencias externas o de Spring, forzando fallos de compilación mediante el Reviewer Agent si se contamina el directorio.

### Tests del algoritmo (TDD)

- [ ] T103 [P] Test unitario `DimensionesTest`:
  - Validar forzosamente que `new Dimensiones(0, 5, 2)` propague error `DimensionInvalidaException` (Regla > 0).
  - Assert sobre `volumenM3(largo, ancho, alto)`, validando el cálculo en Cms a divisor de 1.000.000 (FR-002).
- [ ] T104 [P] Test unitario `PesoTest`:
  - `new Peso(70.1)` produce error inmediato. Límite de carga cerrado en sistema operativo.
  - Dispara flag `Carga Especial` entre mayores de 50 y <70kg (FR-005).
- [ ] T105 [P] Test unitario `DesgloseTarifarioTest`:
  - Lógica comparativa: El "peso facturable" retiene el mayor número entre `Peso Real` y `Peso Volumétrico` (Volumen x 250).

### Implementación del dominio

- [ ] T106 [P] Construir `Dimensiones` y manejo conceptual abstracto de envolturas:
```java
// backend/domain/model/Dimensiones.java
public class Dimensiones {
    private final Double largoCm;
    private final Double anchoCm;
    private final Double altoCm;
    private final Boolean esFormaIrregular;

    public Dimensiones(Double largo, Double ancho, Double alto, Boolean esFormaIrregular) {
        if (largo <= 0 || ancho <= 0 || alto <= 0) {
            throw new DimensionInvalidaException("Las restricciones geométricas exigen un valor dimensional > 0.");
        }
        // Asignaciones
    }
    
    public Double calcularVolumenM3() { return (largoCm * anchoCm * altoCm) / 1_000_000; }
}
```

---

## Phase 3: Servicio de Aplicación — ProcesarPesajeUseCase (US2)

**Goal:** Ejecución principal que hidrata el identificador del paquete (origen), incrusta los valores tarifarios e invoca los puertos guardando los cálculos inmutables.

### Tests del servicio (TDD)

- [ ] T107 [P] [US2] `ProcesarPesajeUseCaseTest` — Operación Exitosa de Cálculo sin Novedades:
  - Dado: UUID existente de un paquete, configuraciones de Tarifas retornan precios normales.
  - Cuando: el caso de uso orquesta procesar y recibe los DTO limpios.
  - Entonces: Mocks verifican una escritura al `PaqueteRepository.guardar(...)` con precio establecido y estado de dimensiones resueltas.

### Implementación del servicio

- [ ] T108 [P] [US2] Lógica transaccional de Procesamiento y validación:
```java
// backend/application/pesaje/ProcesarPesajeUseCase.java
@Service
@Transactional
public class ProcesarPesajeUseCase {

    private final PaqueteRepository paqueteRepository;
    private final ConfiguracionTarifas tarifas;

    public ProcesarPesajeUseCase(PaqueteRepository paqueteRepository, ConfiguracionTarifas tarifas) {
        this.paqueteRepository = paqueteRepository;
        this.tarifas = tarifas;
    }

    @Override
    public Paquete procesarDimensiones(UUID paqueteId, ComandoRegistroFisico cmd) {
        Paquete paquete = paqueteRepository.buscarPorId(paqueteId); 
        
        Dimensiones dims = new Dimensiones(cmd.largo(), cmd.ancho(), cmd.alto(), cmd.irregular());
        Peso peso = new Peso(cmd.pesoKg());

        TarifaBase tarifasConst = tarifas.getTarifasVigentes(paquete.getFechaIngresoUtc());
        
        PrecioEnvio constEnvio = PrecioEnvio.calcularDesglose(
               peso, dims, paquete.getDistanciaEstimadaKm(), tarifasConst, cmd.tipoMercancia());

        paquete.aplicarCotizacionFisica(peso, dims, constEnvio, cmd.tipoMercancia());
        
        return paqueteRepository.guardar(paquete);
    }
}
```

---

## Phase 4: Adaptadores de Entrada (REST y React UI)

### Adaptador REST (Backend)

- [ ] T109 [US2] Configurar `PesajeController`: Endpoint POST/PUT asociativo (`/api/paquetes/{id}/mediciones`) emitiendo errores 422 Unprocessable Entity en rupturas a nivel validación de datos.

### Aplicación UI (Frontend React)

- [ ] T110 [US2] Construcción UI dinámica en `PesajeDimensionesForm.jsx`. Inclusión de lógica reactiva para mostrar Hojas de Diálogo de Confirmación Forzada visualmente invasivas en caso de que la respuesta HTTP levante la alerta de densidades atípicas (FR-006).
- [ ] T111 [US2] Interfaz con botones de toggle para Activar la característica de "Dimensiones Irregulares", mutando del formulario al operador con imágenes explícitas que limitan y asisten la medición.

---

## Phase N: Polish

- [ ] T112 Refactorización conjunta para garantizar la publicación en Event-Driven: Verificación de que la exitosa confirmación de *Pesaje finalizado* interactúe con el *Workflow* general, procediendo a encender el disparador de Solicitud de Ruta (que permanecía bloqueado por falta de peso).

---

## Dependencies & Execution Order

```text
SPRINT BASE (Entregables anteriores)
    └── MOD1-IP-001 (Admisión)
            └── Este Plan ---> Phase 2, 3, 4
```

---

## Notes

- **Precaución concurrente:** Asegurar la consistencia durante la asignación del peso total.
- **Simuladores:** Requerir simulador estático para el oráculo de Tarifas antes de ir a fase integrativa.

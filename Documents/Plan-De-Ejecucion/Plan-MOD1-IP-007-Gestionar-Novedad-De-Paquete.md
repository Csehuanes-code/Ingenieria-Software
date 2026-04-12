# Implementation Plan: Gestionar Novedad de Paquete (MOD1-IP-007)

**Date:** 2026-04-11
**Spec:** [Gestionar Novedad De Paquete](../Specs/MOD1-UC-007-Gestionar-Novedad-De-Paquete.md)
**Arquitectura:** Ver [Metodologias](../Metodologias.md)
**Orden de ejecución:** Sprint 3 — Enlace integrador entre Módulos Externos (Finanzas / Geocoding-Rutas).

---

## Summary

Opera como el corazón transcriptor de novedades, captando tanto inputs de colas asíncronas desde Rutas en la calle (Parades, PODs de entrega), como imputaciones directas locales. Su obligación cardinal es construir un Bitácora unificada con historial universal trazable, orquestar las notificaciones del cliente, y consolidarse exponiendo un API GET estricto para las confirmaciones atómicas síncronas requeridas obligatoriamente por el exigido Módulo de Gestión de Finanzas.

---

## Technical Context

| Campo | Valor |
|---|---|
| **Language/Version** | Java 21 (Backend) |
| **Primary Dependencies** | Spring Web, Spring Boot Starter AMQP / AWS SQS (Recepcion), PostgreSQL, Java Mail/SMS Integrations |
| **Storage** | PostgreSQL (Control absoluto de historiales multi-módulo) |
| **Testing** | JUnit 5, Mockito, Spring MVC Test (Endpoints Síncronos) |
| **Target Platform** | Servidor Linux (Backend Process Background & Frontend REST API) |
| **Project Type** | Sistema Core de Integración (APIs + Colas de Cola + Workers) |
| **Performance Goals** | Extrema velocidad < 500ms obligatoria en el Endpoint expuesto para el escrutinio de Finanzas (SC-003). |
| **Constraints** | Comunicación Estrictamente Síncrona REST para Finanzas. Bloqueo de duplicados en eventos de M2. |
| **Scale/Scope** | Alto. Intersección funcional prioritaria. Soporte contable y cierres perimetrales. |
| **Framework** | Spring Boot 3.x (Backend) |
| **Arquitectura** | Hexagonal (Ports & Adapters) — Backend Aislado transaccional |
| **Mensajería** | Cola asíncrona de Entrada (M2 -> M1); API REST para Salida (M1 -> FINANZAS) |

---

## Project Structure

> La arquitectura contempla puertos duales: Los REST Web clásicos (Input de Controllers y Consultas) en conjunto a Listeners de Eventos y Puertos Notificadores hacia el usuario humano.

```text
backend/
├── domain/
│   ├── model/
│   │   ├── Paquete.java                       [MODIFICADO]
│   │   ├── HistorialEstadoGlobal.java         [NUEVO — Sustituto maduro universal Inmutable]
│   │   └── EventoTransito.java                [NUEVO - Value Object representativo temporal]
│   └── external/
│           ├── HistorialRepositorio.java      [NUEVO]
│           └── NotificadorSmsEmail.java       [NUEVO]
│
├── application/
│   └── historiales/
│       └── TransaccionesHistoricasUseCase.java [NUEVO — Orquestador]
│
└── infrastructure/
    ├── adapter/
    │   ├── in/
    │   │   ├── web/
    │   │   │   └── FinanzasQueryController.java [NUEVO — Puerto Síncrono exigido GET /route/../package]
    │   │   └── messaging/
    │   │       └── NovedadExternaConsumer.java[NUEVO — Listener M2]
    │   └── out/
    │       ├── persistence/
    │           └── HistorialJpaAdapter.java   [NUEVO]
    │       └── notification/
    │           └── TwilioAwsSesNotificador.java[NUEVO]
    └── dto/
        └── response/
            └── EstadoPaqueteFinanzasDto.java  [NUEVO — Payload financiero SC-003]
```

---

## Phase 1: Prerequisitos 

- [ ] T101 Disponer configuraciones en application.yml para SMTP u opciones Serverless (AWS SES / SendGrid) para correos y pasarelas de SMS (FR-002).

---

## Phase 2: Dominio de Unificación Histórica y Trazas

**Purpose:** Validar, impedir duplicidades de timestamps e invalidación inmutable para sustentar a Finanzas.

### Tests del algoritmo (TDD puro)

- [ ] T102 [P] Test unitario `HistorialEstadoGlobalTest`:
  - `agregarAHistorial(...)` rechaza un intento si el ID del Evento M2 suministrado coincide con lo preexistente en la validación contra UUID de red (Deduplicación / FR-008). 

### Implementación del dominio

- [ ] T103 [P] Entidades Inmutables robustas:
```java
// backend/domain/model/HistorialEstadoGlobal.java
public class HistorialEstadoGlobal {
    private final UUID operacionId;
    private final UUID paqueteRelacionado;
    private final String origenTrigger; // 'BODEGA', 'M2_RUTAS'
    
    // deduplicators
    private final String externalReferenceMessageId;

    public boolean esDuplicado(String otherId) {
        return this.externalReferenceMessageId.equals(otherId);
    }
}
```

---

## Phase 3: Servicio de Aplicación — TransaccionesHistoricasUseCase (US7)

**Goal:** Proveer lógica asertiva, canalizando la re-invocación a notificaciones del usuario en base al nivel crítico del estado, y proveyendo un Query puro e indexado del Paquete.

### Tests del servicio (TDD)

- [ ] T104 [P] [US7] Control Anti-Duplicado y Orquestación:
  - Verificar que si el `Consumer` inyecta un payload cuyo ID M2 ya fue guardado en un Historial Previo, la Base de Datos transaccional y el `UseCase` lo omiten, no cambian estado, y retornan ejecución limpia (Idempotencia). 

### Implementación del servicio

- [ ] T105 [P] [US7] Aplicar orquestación síncrona / asíncrona dual:
```java
// backend/application/historiales/TransaccionesHistoricasUseCase.java
@Service
@Transactional
public class TransaccionesHistoricasUseCase {
    private final PaqueteRepository paquetes;
    private final HistorialRepositorio histogramA;
    private final NotificadorSmsEmail alertaVisual;

    @Override
    public void incorporarTrazaLogisticaExterna(ComandoLlegadaRuta msg) {
        if(histogramA.existeIdMensajePrevio(msg.getMessageId())) return; // Deduplicate
        
        Paquete pq = paquetes.buscarPorId(msg.getPaqueteId());
        
        HistorialEstadoGlobal nuevoRegistro = HistorialEstadoGlobal.derivar(msg, pq);
        histogramA.guardar(nuevoRegistro);
        
        pq.forzarEstado(msg.getNuevoEstadoSugerido());
        paquetes.guardar(pq);
        
        alertaVisual.enviarSmsYMail(pq.getDestinatarioTelemetria(), pq.getDestinatarioMail(), "Alerta Actualización"); 
        // FR-002 Notifica
    }
}
```

---

## Phase 4: Adaptadores (REST Público y Colas M2)

### Receptor Asíncrono de Rutas (M2)

- [ ] T106 [US7] Instanciar `RabbitListener` / `SqsListener` procesando las colas predeterminadas (ej. `package-transit-updates`) emitiendo consumps hacia la base (incorporarTraza).

### Interfaz Rest Consultiva Síncrona para FINANZAS

- [ ] T107 [US7] Exigencia Crítica Financiera. `FinanzasQueryController.java`.
  - Crear endpoint mandatorio `GET /route/{idRoute}/package/{idPaquete}`.
  - Implementar consulta SQL plana rápida optimizada por `@Query` en Repository devolviendo DTO crudo (`EstadoPaqueteFinanzasDto`) evitando ORM mappings profundos para mantener el tiempo latente estrictamente por debajo de los 500ms según mandato en Criterios de Éxito (FR-005, FR-006, SC-003). Invocará código 404 (FR-007) si DB no asiste el paquete.

---

## Phase N: Polish

- [ ] T108 Testing estresante de Carga en JMeter o Gatlind para endpoint `/route/../package` para sostener latencias por requerimientos auditables, usando PostgreSql Indices combinados (`índice compuesto route_id + paq_id`).

---

## Dependencies & Execution Order

```text
SPRINT BASE
    └── MOD1-IP-001 / MOD1-IP-003
            └── Este Plan ---> Phase 2, 3, 4
```

---

## Notes

- **Exigencia de latencia:** Cumplir el umbral <500ms al consultar Finanzas. Se prohíbe ORM pesado en esta traza de salida.

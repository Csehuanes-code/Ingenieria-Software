# Implementation Plan: Solicitar Ruta de Paquete (MOD1-IP-003)

**Date:** 2026-04-11
**Spec:** [Solicitar Ruta De Paquete](../Specs/MOD1-UC-003-Solicitar-Ruta-De-Paquete.md)
**Arquitectura:** Ver [Metodologias](../Metodologias.md)
**Orden de ejecución:** Sprint 2 — Sujeto al evento detonedor de creación y pesado.

---

## Summary

La comunicación estricta y asíncrona mediante APIs de Eventos. Provee el componente intermedio que, tan pronto detecta que el paquete está registrado y costeado (pesado), inyecta de forma garantizada su payload unívoco hacia el Módulo de Gestión de Rutas (o una cola abstracta SQS). Recibe recíprocamente un identificador (`ID de Ruta`) para asignárselo a la inmutabilidad propia en Base de Datos central del Módulo de Paquetes. Utiliza Java 21, Spring Boot y patrones Event-Driven de Arquitectura Limpia.

---

## Technical Context

| Campo | Valor |
|---|---|
| **Language/Version** | Java 21 (Backend) |
| **Primary Dependencies** | Spring Web, Spring Boot Starter AMQP / AWS SQS, Jackson (Mapeo JSON), PostgreSQL |
| **Storage** | PostgreSQL (Grabación eventual del ID de Ruta sobre entidades) |
| **Testing** | JUnit 5, Mockito, Testcontainers |
| **Target Platform** | Servidor Linux (Backend Process - Background Queue Consumer/Publisher) |
| **Project Type** | Web Service Intercomunicado vía Mensajes (Colas) |
| **Performance Goals** | Inyección a la cola o procesamiento sub-segundo; Retries asíncronos sin bloqueo de red front. |
| **Constraints** | Restricción temporal y estricta: Las solicitudes se generan una sola vez. JSON Payload estático. |
| **Scale/Scope** | Alto impacto. Acoplamiento logístico entre sub-sistemas y tolerancia a fallos en Módulo Externo. |
| **Framework** | Spring Boot 3.x (Backend) |
| **Arquitectura** | Hexagonal (Ports & Adapters) — Enfoque Event-Driven |
| **Mensajería** | Amazon SQS / RabbitMQ |

---

## Project Structure

> Aislamiento Hexagonal. Se enfatiza el rol pasivo del receptor.

```text
backend/
├── domain/
│   ├── model/
│   │   └── SolicitudRutaRecord.java           [NUEVO — Registro interno de auditoría y resultado]
│   ├── exception/
│   │   └── DuplicateRouteSolicitudeException.java [NUEVO]
│   └── external/
│           ├── RutaEventPublisher.java        [MODIFICADO — Instanciado en IP-001]
│           └── LogRutaRepository.java         [NUEVO]
│
├── application/
│   └── ruta/
│       └── AsignarRutaInboundUseCase.java     [NUEVO — Orquestador de lógica combinada]
│
└── infrastructure/
    ├── adapter/
    │   ├── in/
    │   │   └── messaging/
    │   │       └── ReceiverAsignacionConsumer.java [NUEVO — Listener que escucha el nuevo ID]
    │   └── out/
    │       └── messaging/
    │           └── SqsRutaEventAdapter.java   [NUEVO — La implementación física de envío a la cola M2]    
    └── dto/
        └── event/
            ├── PayloadIdRutaReceptor.java     [NUEVO]
            └── SolicitudAsignacionMessage.java[NUEVO]
```

---

## Phase 1: Prerequisitos (verificación e infraestructura de red)

- [ ] T101 Configuración en Infraestructura de las credenciales, Secret Keys o URIs de los Message Brokers seleccionados en el `.env` para la conectividad SQS/AMQP.
- [ ] T102 Garantía en la migración de DB (`Flyway`) sobre el soporte estricto del campo `ruta_id` en la tabla `paquetes` y creación de tabla para la bitácora `solicitud_ruta_logs`.

---

## Phase 2: Dominio Lógico Integral de Eventos 

**Purpose:** Validar semánticamente la imposibilidad de re-disparos e inyectar inmutabilidad a los registros informativos.

### Tests del algoritmo (TDD puro)

- [ ] T103 [P] Test unitario `PaqueteTest` (Actualización):
  - Inyectar lógica de seguro: Un `Paquete` que llama a método `asignarIdDeRuta(...)` bloqueará una subsiguiente llamada si `ruta_id` no es Null originando un `DomainException`.
  - Asegurar incapacidad de emitir si se detecta falta de peso y dimensiones en el conjunto de origen.

### Implementación del dominio

- [ ] T104 [P] Ajustes de control al dominio primario de Paquete existente:
```java
// backend/domain/model/Paquete.java - (Extrayendo logica)
public void asignarIdentificadorExternoRuta(UUID idRutaExterna) {
    if(this.rutaId != null) {
        throw new EstadoInvalidoException("La solicitud de transporte asíncrono no puede ser sobreescrita ni re-emitida bajo ID existences.");
    }
    this.rutaId = idRutaExterna;
}
```

---

## Phase 3: Servicio de Aplicación — AsignarRutaInboundUseCase (US3)

**Goal:** Proporcionar la lógica que reacciona tanto a envíos hacia la salida, como a la actualización producida por la entrada desde las colas de respuesta.

### Tests del servicio (TDD)

- [ ] T105 [P] [US3] Procesar Respueta asíncrona de M2:
  - Dado: Mensaje JSON válido derivado en cola y mapeado al sistema, simulando al M2 despachando la asignación.
  - Cuando: el consumer activa `AsignarRutaInboundUseCase.completarAsignacion(...)`.
  - Entonces: Módulo hidrata el `PaqueteDb`, verifica ausencia previa de ID y guarda actualizando el campo atómicamente loggeandolo en `SolicitudRutaRecord`.

### Implementación del servicio

- [ ] T106 [P] [US3] Implemetar control transaccional de recepción y mutación de estado:
```java
// backend/application/ruta/AsignarRutaInboundUseCase.java
@Service
@Transactional
public class AsignarRutaInboundUseCase {
    private final PaqueteRepository paqueteRepository;
    // ...

    public void completarAsignacionAsincrona(UUID paqueteId, UUID idRutaProveniente) {
        Paquete pq = paqueteRepository.buscarPorId(paqueteId); // Fetch
        
        pq.asignarIdentificadorExternoRuta(idRutaProveniente); // Falla si intenta doble inyección
        
        paqueteRepository.guardar(pq);
        // Desencadena lógica complementaria como avisar mediante SocketUI si procede...
    }
}
```

---

## Phase 4: Adaptadores de Mensajería (REST Oculto / Colas)

### Interfaz Inbound (Recibidor)

- [ ] T107 [US3] Implementar Spring Messaging (`SqsListener` o `@RabbitListener`) en `ReceiverAsignacionConsumer.java`.
  - El consumer atiende la cola designada leyendo el Payload de respuesta en JSON que manda la ruta asignada y lo pre-procesa o parsea mediante Jackson Object Mappers.

### Interfaz Outbound (Productor)

- [ ] T108 [US3] Implementar puerto de salida concreto `SqsRutaEventAdapter.java`. Su tarea recae en la creación del JSON estático: "UUID del paquete, peso, volumen, tipo mercancia, dirección, y coordenadas GPS" (FR-002) que el M2 exige, empujándolo agresivamente contra el bróker asícrono tras el evento `PesajeFinalizado`.

---

## Phase N: Polish

- [ ] T109 Configuración estricta en infraestructura (`application.properties`) previniendo políticas infinitas de reconectividad o "dead letter queues" en caso de falla terminal del M2 (Timeout loggeo asíncrono para TDD en Phase 1/3).

---

## Dependencies & Execution Order

```text
SPRINT BASE
    └── MOD1-IP-001 / MOD1-IP-002
            └── Este Plan ---> Phase 2, 3, 4
```

---

## Notes

- **Precaución asíncrona:** Asegurar el manejo correcto en caso de que el mensaje entrante provenga de fallos de red encolados.

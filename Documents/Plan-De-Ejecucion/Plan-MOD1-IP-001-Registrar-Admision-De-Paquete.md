# Implementation Plan: Registrar Admisión de Paquete (MOD1-IP-001)

**Date:** 2026-04-10 (actualizado)
**Spec:** [Registrar Admision De Paquete](../Specs/MOD1-IP-001-Registrar-Admision-De-Paquete.md)

---

## Summary

Como Empleado de Envío y Recepción, necesito registrar los datos del remitente, destinatario y paquete para generar su identidad en el sistema y disparar la solicitud de ruta al completarse el pesaje. El sistema se gestionará mediante React y Spring Boot (Java 21) estructurados en una Arquitectura Hexagonal, garantizando validaciones de Geocoding rápidas (<5s), persistencia centralizada en PostgreSQL e integraciones asíncronas vía colas de mensajes (SQS/AMQP) para los disparadores hacia el módulo de rutas.

---

## Technical Context

| Campo | Valor |
|---|---|
| **Language/Version** | Java 21 (Backend) / JavaScript (React para Frontend) |
| **Primary Dependencies** | Spring Web, Spring Data JPA, Spring Security, Spring Validation, PostgreSQL, Google Maps Services, Spring Boot Starter AMQP / AWS SQS, Gradle |
| **Storage** | PostgreSQL (control transaccional del UUID e historial inmutable) |
| **Testing** | JUnit 5, Mockito, Testcontainers (PostgreSQL) |
| **Target Platform** | Servidor Linux para Backend API y Web browser para la UI |
| **Project Type** | Single Web Application (Backend / Web) |
| **Performance Goals** | Respuesta de Geocoding < 5s; Transacciones web sin bloqueos asíncronos prolongados |
| **Constraints** | UUID unico inmutable. Fallback obligatorio manual ante fallos de APIs externas. |
| **Scale/Scope** | Módulo Core principal de entrada. Soporte para alta concurrencia inicial de paquetes recibidos por sucursal. |
| **Framework** | Spring Boot 3.x (Backend), React (Frontend) |
| **Arquitectura** | Hexagonal (Ports & Adapters) — Backend aislado |
| **Mensajería** | Cola asíncrona (Event-Driven) para notificar `solicitar_ruta` |

---

## Project Structure

> Se usa la estructura hexagonal definida en las directrices. Los archivos de este plan identifican dónde va cada clase backend y sus contrapartes en frontend.

```text
frontend/
└── src/
    ├── components/
    │   └── form/
    │       └── AdmisionPaqueteForm.jsx      [NUEVO — Componente principal React UI]
    └── services/
        └── AdmisionApiService.js            [NUEVO — HTTP Fetchs a Spring Boot]

backend/
├── build.gradle                             [MODIFICADO — Inclusión de dependencias]
├── domain/
│   ├── model/
│   │   ├── Paquete.java                     [NUEVO]
│   │   ├── Sede.java                        [NUEVO]
│   │   └── Persona.java                     [NUEVO — Remitente/Destinatario]
│   ├── valueobject/
│   │   ├── Direccion.java                   [NUEVO]
│   │   └── Coordenadas.java                 [NUEVO]
│   ├── exception/
│   │   ├── InvalidCoverageException.java    [NUEVO]
│   │   └── TimeoutGeocodingException.java   [NUEVO]
│   └── external/
│           ├── PaqueteRepository.java       [NUEVO]
│           ├── GeocodingService.java        [NUEVO]
│           └── RutaEventPublisher.java      [NUEVO]
│
├── application/
│   └── admision/
│       └── RegistrarAdmisionUseCase.java    [NUEVO — Orquestador de lógica de negocio]
│
└── infrastructure/
    ├── adapter/
    │   ├── in/
    │   │   └── web/
    │   │       └── AdmisionController.java  [NUEVO — Endpoints consumidos por React]
    │   └── out/
    │       ├── persistence/
    │       │   ├── PaqueteJpaAdapter.java   [NUEVO]
    │       │   └── PaqueteJpaRepository.java
    │       ├── external/
    │       │   └── GoogleMapsAdapter.java   [NUEVO — HTTP client con timeout 5s]
    │       └── messaging/
    │           └── RutaEventAdapter.java    [NUEVO — EventPublisherPort para colas]
    └── dto/
        ├── request/
        │   └── RegistroAdmisionRequest.java [NUEVO]
        └── response/
            └── RegistroAdmisionResponse.java [NUEVO]
```

---

## Phase 1: Prerequisitos (verificación)

> **Dependencia bloqueante:** Configuración inicial del repositorio de Gradle y empaquetado de React/Node.

- [ ] T101 Verificar que `build.gradle` tenga las dependencias explícitas de Java 21, Spring Boot Web, Spring Data JPA, Driver PostgreSQL de manera correcta.
- [ ] T102 Verificar disponibilidad del archivo `.env` o `application.yml` incluyendo la API Key de Google Maps y las credenciales de PostgreSQL / RabbitMQ / AWS.
- [ ] T103 Validar conectividad de Base de Datos local mediante Testcontainers y confirmar acceso del script `Flyway`/migración de creación de la tabla principal `paquetes`.

---

## Phase 2: Dominio de Admisión — Lógica y Validaciones
**Purpose:** La lógica pura permanece aislada, garantizando las reglas sin verse afectadas por las tecnologías de base o mensajería web.

⚠️ **CRÍTICO: Ningún archivo o clase bajo la carpeta `domain/` puede contener importaciones de Spring o librerías externas que no sean propias de Java 21.**

### Tests del algoritmo (TDD puro)

- [ ] T104 [P] Test unitario `PaqueteTest`:
  - Instanciar `new Paquete(...)` auto-genera su UUID `v4` inmutable y fecha actual UTC.
  - Genera el paquete en estado obligatorio `RECIBIDO_EN_SEDE`.
- [ ] T105 [P] Test unitario `SedeTest`:
  - `validarMetodoPagoSoportado(...)`: confirma si la Sede permite métodos Contra Entrega o Prepago según especificación de requerimientos de esa sede específica.

### Implementación del dominio

- [ ] T106 [P] Construir `Paquete`, garantizando UUIDs v4, y estado inmutable de fecha de registro:
```java
// backend/domain/model/Paquete.java
public class Paquete {
    // Identidad y Estado General (MOD1-UC-001)
    private final UUID id;
    private final LocalDateTime fechaIngresoUtc;
    private EstadoPaquete estado;
    private SedeId sede;

    // Destino y Valor (MOD1-UC-001)
    private final Direccion direccionDestino;
    private Coordenadas coordenadas;
    private EstadoGps estadoGps;
    private BigDecimal valorDeclarado;
    private MetodoPago metodoPago;
    private Persona remitente;
    private Persona destinatario;

    // Atributos Físicos y Tarifarios (MOD1-UC-002)
    private Peso peso;
    private Dimensiones dimensiones;
    private Double volumenM3;
    private Double pesoVolumetrico;
    private Double pesoFacturable;
    private TipoMercancia tipoMercancia;
    private CategoriaCarga categoriaCarga;
    private Boolean indicadorFormaIrregular;
    private PrecioEnvio precioEnvio; // Inmutable tras el pesaje
    private Double distanciaEstimadaKm;

    // Asignación de Rutas y Zonas (MOD1-UC-003, UC-004, UC-005)
    private UUID rutaId; // Asignado asíncronamente por Módulo 2
    private UUID zonaAlmacenamientoId; // Física en bodega
    private UUID zonaDestinoId; // Clasificación lógica para ruteo

    public Paquete(SedeId sede, Direccion direccionDestino, Persona remitente, Persona destinatario) {
        this.id = UUID.randomUUID();
        this.fechaIngresoUtc = LocalDateTime.now(ZoneOffset.UTC);
        this.estado = EstadoPaquete.RECIBIDO_EN_SEDE;
        this.sede = sede;
        this.direccionDestino = direccionDestino;
        this.remitente = remitente;
        this.destinatario = destinatario;
        this.estadoGps = EstadoGps.PENDIENTE;
    }
    
    public void asignarCoordenadas(Coordenadas latLon) {
        this.coordenadas = latLon;
        this.estadoGps = EstadoGps.RESUELTO;
    }
}
```

---

## Phase 3: Servicio de Aplicación — RegistrarAdmisionUseCase (US1)

**Goal:** Orquestador de lógica de negocio, consumiendo puertos. Recibe los comandos del controller web, valida y persiste transaccionalmente.

**Independent Test:** Instanciar `RegistrarAdmisionUseCase` usando implementaciones Mock de las dependencias externas.

### Tests del servicio (TDD)

- [ ] T107 [P] [US1] `RegistrarAdmisionUseCaseTest` — Registro Geocoding Exitoso:
  - Dado: Payload del Paquete válido, el servicio de Geocoding retorna lat/lon, pesaje en la solicitud exitoso.
  - Cuando: Se ejecuta el caso de uso registrar().
  - Entonces: `PaqueteRepository.guardar()` se efectúa y `RutaEventPublisher.publicarSolicitudRuta()` es disparado.

- [ ] T108 [P] [US1] `RegistrarAdmisionUseCaseTest` — Mapeo Fallido Timeout API:
  - Dado: `GeocodingService` lanza una alerta o timeout por tardanza (+5s).
  - Cuando: Se solicita registrar.
  - Entonces: El paquete se guarda pero **el envío asíncrono para Rutas** NO se emite sin antes haber resuelto de forma manual la coordenada.

- [ ] T109 [P] [US1] `RegistrarAdmisionUseCaseTest` — Fuera de Rango:
  - Dado: Respuesta de Geocoding confirma falta de servicio para zona dictaminada.
  - Cuando: Se intenta asentar.
  - Entonces: Lanza al cliente excepción/error `InvalidCoverageException` con interrupción agresiva de proceso.

### Implementación del servicio

- [ ] T110 [P] [US1] Implementar caso de uso bajo `@Service` e inyección de dependencia obligatoria por constructor:
```java
// backend/application/admision/RegistrarAdmisionUseCase.java
@Service
@Transactional
public class RegistrarAdmisionUseCase {

    private final PaqueteRepository paqueteRepository;
    private final GeocodingService geocodingService;
    private final RutaEventPublisher eventPublisher;

    public RegistrarAdmisionUseCase(PaqueteRepository paqueteRepository, 
                                    GeocodingService geocodingService,
                                    RutaEventPublisher eventPublisher) {
        this.paqueteRepository = paqueteRepository;
        this.geocodingService = geocodingService;
        this.eventPublisher = eventPublisher;
    }

    @Override
    public UUID registrarAdmision(RegistroAdmisionCommand command) {
        Coordenadas coordenadas = null;
        try {
            coordenadas = geocodingService.localizar(command.direccion());
        } catch (TimeoutGeocodingException e) {
            // El fallback: El empleado deberá llenar las coordenadas a mano 
            // a través de React posteriormente al guardado parcial.
        }

        Paquete paquete = new Paquete(command.direccion());
        if (coordenadas != null) {
            paquete.asignarCoordenadas(coordenadas);
        }
        
        Paquete saved = paqueteRepository.guardar(paquete);

        if (coordenadas != null && haEjecutadoPesaje(command)) {
             eventPublisher.publicarSolicitudRuta(saved.getId());
        }

        return saved.getId();
    }
}
```

---

## Phase 4: Adaptadores de Entrada (REST y React)

### Adaptador REST (Backend)

- [ ] T111 [US1] Configurar `AdmisionController` implementando validaciones Spring Bean para garantizar limpieza de nulls y tipos correctos que entran desde la UI. 
- [ ] T112 [US1] Definir excepción en Controller Advice para tratar `InvalidCoverageException` y retornar HTTP Status comprensible para el Frontend (ej. estado 400 ó 422).

### Aplicación UI (Frontend Javascript/React)

- [ ] T113 [US1] UI Component `AdmisionPaqueteForm.jsx`: Maquetar un formulario claro para el empleado garantizando alertas visuales para "Fuera de Cobertura" basado en la respuesta HTTP.
- [ ] T114 [US1] Componente dinámico React que responda visualmente ocultando/mostrando el fragmento de "Ingresar Coordenadas Manualmente" solo cuando la validación de Geocoding retorna un aviso pertinente de timeout o fallback (FR-009).

### Adaptador Mensajería (Backend)

- [ ] T115 [US1] Implementación `GoogleMapsAdapter` garantizado que se corte estrictamente por timeout limit de Spring si la HTTP request natural a Google Maps pasa de los 5,000 milisegundos.

---

## Phase N: Polish

- [ ] T116 Validar consistencia global, asegurando que todos los tests ejecutan pasando en un GitHub Action (CI de Gradle) y CI de frontend (Vitest / Jest).
- [ ] T117 Test de integración transaccional que emule la llamada REST hasta el commit a base de datos PostgreSQL, asegurando la no interferencia de transacciones paralelas.

---

## Dependencies & Execution Order

```text
SPRINT BASE (Spring Init, Java 21 SDK, Vite+React init)
    └── Este Plan: Módulo Principal de Admisiones
            ├── Phase 2 (Dominio: Entidades y validadores)
            ├── Phase 3 (Servicio orquestador transaccional en application)
            ├── Phase 4 (Construcción del Controller y Forms de React UI)
            └── Phase N (Flujos de CI/CD para pulido)
```

---

## Notes

- **Separación de pre-requisitos de SGP**: Tener precaución sobre variables `haEjecutadoPesaje()`, predecir un simulador para la fase mientras se realiza la interacción con el módulo 2 de dimensiones corporales.
- **Transmisión de UUIDs asíncrona**: Mantener a salvo la llave foránea sobre `solicitar_ruta` verificando explícitamente en base de datos.

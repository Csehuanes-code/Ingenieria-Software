# Implementation Plan: Actualizar Estado de Paquete por Novedad (MOD1-IP-006)

**Date:** 2026-04-11
**Spec:** [Actualizar Estado de Disponibilidad](../Specs/MOD1-UC-006-Actualizar-Estado-De-Disponibilidad.md)  *(Título: Actualizar Estado de Paquete por Novedad)*
**Arquitectura:** Ver [Metodologias](../Metodologias.md)
**Orden de ejecución:** Sprint 3 — Flujo alternativo (Excepción).

---

## Summary

Inyectar un modelo rastreable e interbloqueante para la gestión de novedades (Dañado/Extraviado) surgidas internamente en bodega. Facilita una subida multimedia (Imágenes/Vídeos) obligando su presencia desde una aplicación SPA de React acoplada a un servidor estático (S3). El backend en Spring Boot resguardará un riguroso e inmutable Bitácora de Transiciones validando que ningún producto ya embarcado en camiones pueda ser falsamente reportado en bodega.

---

## Technical Context

| Campo | Valor |
|---|---|
| **Language/Version** | Java 21 (Backend) / JavaScript (React para Frontend) |
| **Primary Dependencies** | Spring Web, Spring WebFlux/Multipart, PostgreSQL, AWS S3 API (Media), Hibernate |
| **Storage** | PostgreSQL (Tabla `historial_novedades`); Blob Storage (Videos/Imágenes de Evidencia) |
| **Testing** | JUnit 5, Mockito, WireMock (para emulación subidas S3/Minio) |
| **Target Platform** | Servidor Linux para Interfaz Rest y Blob Storage / Navegador Móvil/Tablet en PDA Bodega |
| **Project Type** | Web application (Multi-Multipart Form y APIs) |
| **Performance Goals** | Subidas de archivos en back-pressure streams < 15 segundos sin ahogar el Application Server. |
| **Constraints** | Bloqueo absoluto de creación de daño sin evidencia multimedia o con estado superior a Tránsito. |
| **Scale/Scope** | Mediano alcance pero Alta responsabilidad judicial / financiera (Soporte contable de seguros). |
| **Framework** | Spring Boot 3.x (Backend), React (Frontend) |
| **Arquitectura** | Hexagonal (Ports & Adapters) — Servicios de Eventos y Multimedia separados |
| **Mensajería** | Webhooks locales o Event Bus local hacia el Módulo del IP-007 (Control de Trazabilidad) |

---

## Project Structure

> Aislamiento estricto de las Entidades de la Bitácora inmutable fuera del modelo principal acoplado del Paquete.

```text
backend/
├── domain/
│   ├── model/
│   │   ├── Paquete.java                       [MANTIENE LA ESTRUCTURA PRINCIPAL]
│   │   ├── NovedadBodega.java                 [NUEVO — Registro aislado y perenne]
│   │   └── MultimediaEvidencia.java           [NUEVO — Value object ref a URIs de almacenamiento]
│   ├── exception/
│   │   ├── EstadoNoReversibleException.java   [NUEVO]
│   │   └── EvidenciaRequeridaException.java   [NUEVO]
│   └── external/
│           ├── FileStorage.java               [NUEVO — Puerto ciego contra CDN o S3]
│           └── HistorialNovedadesRepository.java[NUEVO]
│
├── application/
│   └── novedades/
│       └── ReportarNovedadUseCase.java        [NUEVO — Levanta el Storage y salva la DB]
│
└── infrastructure/
    ├── adapter/
    │   ├── in/
    │   │   └── web/
    │   │       └── NovedadBodegaController.java [NUEVO]
    │   └── out/
    │       └── storage/
    │           └── MinioS3FileAdapter.java    [NUEVO — Guarda discos duros/S3]
    └── dto/
        └── request/
            └── CreacionNovedadMultipart.java  [NUEVO — POJO que mapea texto + File]
```

---

## Phase 1: Prerequisitos 

- [ ] T101 Disponer un bucket emulador Minio/S3 dentro del `.yaml` global en contenedores Docker y su configuración respectiva del puerto 9000 para uso de pruebas.

---

## Phase 2: Dominio de Reglas Históricas (Inmutabilidad)

**Purpose:** Establecer restricciones ineludibles al tipo de datos suministrado para reportar Daños y Extravíos.

### Tests del algoritmo (TDD)

- [ ] T102 [P] Test unitario de Dominio `NovedadBodegaTest`:
  - `reportarDanio(...)` lanza un `EvidenciaRequeridaException` (FR-004) en caso de que la lista o cadena de Media contenga un `size() == 0`.
  - Asegurar la asociación unívoca de estampas temporales (Timestamp inmutable) y responsable de operación (ID_Almacenista) asegurando FR-001 y FR-002.
  - Generar el objeto prohíbe retroacciones de aquellos `paquete.getEstado()` transitando o Entregados (FR-006).

### Implementación del dominio

- [ ] T103 [P] Construcción del componente limitante:
```java
// backend/domain/model/NovedadBodega.java
public class NovedadBodega {
    private final UUID idTransicion;
    private final UUID paqueteIdBase;
    private final String userIdAlmacenista;
    private final LocalDateTime fechahoraGeneracionUTC;
    
    // Lista controlable
    private List<MultimediaEvidencia> evidencias;
    // ...
    
    public NovedadBodega(UUID uuidBase, TipoNovedadLocal tipo, List<MultimediaEvidencia> media, String user) {
        if(tipo == TipoNovedadLocal.DANADO && media.isEmpty()) {
            throw new EvidenciaRequeridaException("Toda denuncia de daño físico amerita mínimo una fuente visual de sustento.");
        }
        // ... set properties
    }
}
```

---

## Phase 3: Servicio de Aplicación — Multi-Port Report (US6)

**Goal:** Recepcionar la stream binaria de la imagen, despacharla al servidor de archivos e inyectar el String indexado dentro del caso formal hacia base de datos relacional.

### Implementación del servicio

- [ ] T104 [P] [US6] Orquestación:
```java
// application/novedades/ReportarNovedadUseCase.java
@Service
@Transactional
public class ReportarNovedadUseCase {
    private final PaqueteRepository paquetes;
    private final HistorialNovedadesRepository history;
    private final FileStorage fileStore;
    // ...

    @Override
    public NovedadBodega someterNovedad(UUID paqueteId, FormatoMulti multipartParams) {
        Paquete paqueteContexto = paquetes.buscarParaUpdate(paqueteId); // Lock Optimista contra otro almacenista
        
        if (paqueteContexto.estaEnTransitoLogisticoFinal()) {
            throw new EstadoNoReversibleException(); // FR-006 Bloquea a los En transito 
        }

        List<MultimediaEvidencia> listUrlsCdn = new ArrayList<>();
        if(!multipartParams.getFiles().isEmpty()){
            // Bloqueante S3/Subida, considerar hacer eventuales las fotos asíncronamente si el FileStore admite promesa
            listUrlsCdn = fileStore.upload(multipartParams.getFiles()); 
        } 
        
        NovedadBodega record = new NovedadBodega(paqueteId, multipartParams.tipo(), listUrlsCdn, multipartParams.usuario());
        history.persistirRegistroInmutable(record);
        
        paqueteContexto.setEstado(EstadoPaquete.NOVEDAD_LOCAL);
        paquetes.guardar(paqueteContexto);
        
        // El Notificador de Sistema (EventBus local) alerta al Controlador (FR-005)
        return record;
    }
}
```

---

## Phase 4: Adaptadores de Interfaz REST Multipart y Frontend

### Adaptador REST (Backend)

- [ ] T105 [US6] Configurar una ruta POST de formato `multipart/form-data`. Controlar de forma absoluta (vía WebMvcConfigurer o Properties globales de Tomcat) el MAX_FILE_SIZE para prevenir DDOS mediante el colapso de Buffers.

### Interfaz UI React Frontend

- [ ] T106 [US6] Utilización del DOM para FileUpload Dropzones previsualizando Imágenes seleccionadas en Thumbnails con base64 antes de emitir y ahogar la red. Controlar y bloquear el botón "Guardar" condicionalmente a que el State posea datos binarios adjuntos caso contrario el API respondería 400 Bad Request.

---

## Phase N: Polish

- [ ] T107 Pruebas Unitarias/Componentes de carga exhaustivas contra `MinioS3FileAdapter` asegurando la limpieza (rollback files) del S3 si el `history.persistirRegistroInmutable()` genera fallo de DB, previniendo basura en el object storage a largo plazo.

---

## Dependencies & Execution Order

```text
SPRINT BASE
    └── MOD1-IP-001 / MOD1-IP-004
            └── Este Plan ---> Phase 2, 3, 4
```

---

## Notes

- **Manejo de archivos:** Asegurar que las desconexiones temporales guardan un estatus temporal de la subida para evitar que el usuario vuelva a cargar fotos pesadas.

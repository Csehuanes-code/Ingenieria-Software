# Guía para la Generación de Planes de Ejecución (Para Agentes IA)

Este documento establece las normativas, criterios y directrices estructurales que los Agentes de Inteligencia Artificial deben seguir estricta y algorítmicamente al momento de interpretar una Especificación (*Spec*) y generar un nuevo documento estructurado de Plan de Ejecución.

---

## 1. Reglas Base de Nomenclatura y Ubicación
1. **Directorio de Destino:** Todo documento generado bajo el concepto de Plan de Ejecución o Implementación debe persistir de manera obligatoria bajo el directorio `Documents/Plan-De-Ejecucion/`.
2. **Plantilla Base (Template):** La estructura del texto y el esqueleto de las fases deben basarse invariablemente en el archivo matriz `Documents/plan-template.md`.
3. **Archivos de Origen Involucrados:** Los Planes son siempre un derivado tecnológico del caso de uso. El archivo conceptual a analizar previamente descansa en `Documents/Specs/*.md`. La validación metodológica y roles de agentes se derivan de `Documents/Metodologias.md` y `Documents/Agentes.md`.

### Convención de Nombre de Archivos
**Formato Esperado:** `Plan-MOD#-IP-###-[Nombre].md`

**Pasos de transformación que debe aplicar el Agente:**
- Aislar el identificador unívoco proveniente del documento original (Ej. Para el archivo base `MOD1-UC-001-Registrar-Admision-De-Paquete.md`).
- Transmutar el elemento lógico de Casos de Uso `UC` (Use Case) hacia `IP` (Implementation Plan), obteniendo el prefijo `Plan-MOD1-IP-001`.
- Concatenar en estricto _kebab-case_ el nombre o título de la funcionalidad.
- **Resultado Ejemplar:** `Plan-MOD1-IP-001-Registrar-Admision-De-Paquete.md`.

---

## 2. Descripción de las Secciones del Documento de Plan

El Agente debe ser capaz de consumir el Spec y completar eficientemente los siguientes módulos del marco:

### 2.1. Summary
- **Sintaxis:** Texto conciso (1 a 2 párrafos como máximo).
- **Contenido Esperado:** El Agente debe extractar de forma precisa el *Primary Requirement* del archivo *Spec*. Inmediatamente después, debe definir el *Technical Approach*, especificando qué Stack y Arquitectura liderarán el diseño lógico (ejs. React para el UI, Spring Boot Java 21, Arquitectura Hexagonal/Clean, tipo y rol de Base de Datos y enfoques asíncronos si existen colas).

### 2.2. Technical Context
- **Sintaxis:** Tabla Markdown pura (`| Campo | Valor |`).
- **Contenido Esperado:** Completar obligatoria y textualmente los siguientes campos extraídos del análisis de entorno:
  - `Language/Version`
  - `Primary Dependencies`
  - `Storage`
  - `Testing`
  - `Target Platform`
  - `Project Type`
  - `Performance Goals`
  - `Constraints`
  - `Scale/Scope`
- Se deben anexar filas adicionales como `Framework`, `Arquitectura` y enfoques de `Mensajería` donde aplique para ofrecer un contexto global robusto.

### 2.3. Project Structure
- **Sintaxis:** Bloque Markdown en texto crudo (` ```text `) simulando un esquema de árbol.
- **Contenido Esperado:** Debe representar una simulación precisa del despliegue eventual de los repositorios. Obligatoriamente, esta área debe marcar la segregación, visualizando la capa core del `domain/` aislada. Las dependencias externas (Bases de datos, Mensajería, APIs) se designan estricta y únicamente en la subcarpeta `domain/external/` sin sufijos de tipo "Port". Los Casos de Uso de lógica de negocio fungen como orquestadores concretos y reposan directo bajo `application/` (No crear puertos `in/` ni referirse a ellos como "Service"). Se indicará entre corchetes el estatus de la clase (Ej. `[NUEVO]`).
- **Infraestructura y Frontend:** La capa de infraestructura (`infrastructure/`) debe segregar obligatoriamente los adaptadores en `adapter/in/` (ej. `web`) y `adapter/out/` (ej. `persistence`, `external`, `messaging`), e incluir una carpeta `dto/` para objetos request/response. Si el plan incluye UI, debe detallarse también el árbol de `frontend/` (ej. `src/components/`, `src/services/`).

### 2.4. Phase (1-N)
Reflejo del diseño algorítmico y granular de las tareas empleando Test-Driven Development (TDD). 

**Estructura esperada por cada Phase:**
- `Purpose`: Objetivo primario del sprint en la fase actual.
- **Subsección de Pruebas**: Relación de tests explícitos (JUnit, Testcontainers, etc) bajo nomenclatura como `- [ ] TXXX [P] [US1] Unit Test ...`
- **Subsección de Implementación**: Las etapas constructivas, conteniendo fragmentos lógicos de de código (Ej. Java) que le indicarán al Coder Agent cómo ejecutar la manipulación.

**Orden Lógico Frecuente:**
1. **Phase 1: Prerequisitos**, validación rigurosa de dependencias base de build o infraestructura (Ej. validación del Gradle y base de datos).
2. **Phase 2: Dominio**, la declaración de Entidades, Value Objects e interfaces lógicas con sistemas exteriores (estas últimas agrupadas bajo `domain/external/`, omitiendo rigurosamente el sufijo `Port`). Enfoque crudo (Sin frameworks externos).
3. **Phase 3: Casos de Uso (En la Capa Applicación)**, desarrollo de las clases orquestadoras de lógica puramente concretas dentro de `application/` bajo la convención de sufijo `*UseCase.java`. Queda **estrictamente prohibido** utilizar o referenciar el patrón `*Service.java` o el patrón obsoleto de declarar interfaces de entrada en `port/in/`.
4. **Phase 4: Adaptadores de Entrada/Salida**, los controladores Web (Controllers REST), Consumers de mensajes e integraciones con Frontend (React).
5. **Phase N: Polish**, rutinas finales para testing continuo (CI/CD) e integración transaccional global.

### 2.5. Dependencies & Execution Order
- **Sintaxis:** Bloque Markdown en texto crudo (` ```text `) simulando un árbol de ejecución.
- **Contenido Esperado:** Referencia visual estructurada sobre el orden cronológico en que las fases y dependencias externas deben ser abordadas.

### 2.6. Notes
- **Sintaxis:** Lista de viñetas.
- **Contenido Esperado:** Avisos críticos para el Coder Agent, manejo de variables transitorias, simuladores requeridos para interacciones asíncronas, o advertencias sobre bases de datos.

---

## 3. Instrucción Crítica: Estrategia de Búsqueda y Consolidación de Atributos de Entidad
Cuando el Agente esté desarrollando la capa de dominio (`Phase 2`), **no le es permitido limitarse a los campos que indique su Spec primario objetivo**. 

**Metodología de ejecución para el Agente:**
1. Al localizar la Entidad Crítica de análisis (ej. `Paquete`), el agente buscará exhaustivamente la información integral de esta entidad rastreando iterativamente adentro de toda la carpeta `Documents/Specs/`.
2. Para tal fin, el Agente debe hacer uso de las herramientas del sistema (e.g., búsquedas masivas) para localizar en qué otras fases del ciclo de vida ocurren mutaciones o ampliaciones de los objetos relevantes, inspeccionando archivos cruzados.
3. El Agente extraerá atributos adicionales tales como los atributos logísticos, tarifarios o estados extendidos definidos posteriormente por otras dependencias.
4. La Entidad final generada en el plan reflejará la anatomía de madurez máxima del objeto, uniendo atributos lógicos y agrupándolos en comentarios contextualizados dentro del código, certificando un dominio estable.

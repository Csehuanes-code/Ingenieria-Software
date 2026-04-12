# Definición de Agentes del Sistema

Este documento describe la arquitectura de los agentes, detallando la función, la separación de responsabilidades y las estrategias para optimizar el rendimiento de cada uno dentro del ecosistema del proyecto.

## Principios de la Arquitectura de Agentes
1. **Responsabilidad Única (Separation of Concerns)**: Cada agente está diseñado para resolver un dominio cognitivo o funcional específico.
2. **Ejecución Asíncrona**: Las tareas pesadas se encolan y procesan en background para no bloquear el flujo principal.
3. **Alto Rendimiento**: Minimizar el contexto compartido; comunicación basada en mensajes o eventos entre agentes acotando la información inyectada.

---

## 1. Agente Orquestador (Orchestrator Agent)
- **Función**: Actúa como el controlador central y primer punto de contacto. Identifica qué flujo de trabajo se requiere según el prompt del usuario y delega las tareas.
- **Responsabilidades**: 
  - Análisis inicial y enrutamiento a agentes especialistas.
  - Manejo de estados globales de la conversación.
  - Ensamblar la respuesta y presentársela al usuario.
- **Mejora de Rendimiento**: 
  - No realiza procesamiento pesado; mantiene tiempos de respuesta (I/O) bajos.
  - Usa llamadas en paralelo (ej. disparando busquedas concurrentes) y emplea Timeouts rígidos para no quedarse colgado.

## 2. Agente Analista (Analysis & Spec Agent)
- **Función**: Evalúa documentación, requerimientos del usuario y genera un panorama estructurado alineado al Spec Driven Design (SDD).
- **Responsabilidades**:
  - Parseo de los inputs.
  - Investigación y validación de reglas de negocio.
  - Creación de planes de implementación a alto nivel.
- **Mejora de Rendimiento**: 
  - Usa recuperación semántica de contexto (RAG) limitando los chunks inyectados al prompt interno. Evita reprocesar documentos completos mediante un caché de abstracciones (Knowledge Items).

## 3. Agente Ejecutor de Código (Coder/Implementation Agent)
- **Función**: Materializa el diseño en código o modifica estructuras en base a las directivas del Analista.
- **Responsabilidades**:
  - Escritura y modificación asertiva de archivos siguiendo Arquitectura Hexagonal y Limpia.
  - Actualización directa de pruebas unitarias y mocks.
  - Ejecución de comandos del sistema validados.
- **Mejora de Rendimiento**: 
  - Procesamiento enfocado: solo se inyectan en su contexto herramientas enfocadas en manipulación de texto e inputs de terminal, reduciendo la distracción del LLM.
  - Utiliza reemplazos precisos/no-contiguos (diffs directos) en lugar de reescribir los archivos completos, ahorrando tokens de salida drásticamente.

## 4. Agente Revisor y QC (Reviewer Agent)
- **Función**: Es el salvaguarda de calidad que asegura que no haya violaciones arquitectónicas ni regresiones.
- **Responsabilidades**:
  - Auditar que el código del *Coder* no rompa el encapsulamiento de capas (ej: Dominio importando Infraestructura).
  - Validar SCM (mensajes de commit, convenciones de nombres).
- **Mejora de Rendimiento**: 
  - Aplica reglas heurísticas de validación rápida local antes de depender de una costosa llamada al modelo profundo. Solo envía los 'diffs' a analizar, reduciendo enormemente el payload.

## 5. Agente de Monitoreo / Contexto Persistente (Context Agent)
- **Función**: Mantiene un historial conciso de lo actuado entre sesiones.
- **Responsabilidades**:
  - Resumir hilos de conversaciones finalizadas.
  - Consolidar *logs* para la trazabilidad de depuración.
- **Mejora de Rendimiento**:
  - Operación pasiva en tiempo de inactividad, no bloquea al usuario. Comprime datos viejos en *embeddings*, reemplazando la necesidad de historiales textuales infinitos.

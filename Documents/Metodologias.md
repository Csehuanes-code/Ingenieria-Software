# Metodologías y Arquitecturas del Proyecto

Este documento establece las metodologías, patrones y modelos arquitectónicos que regirán el desarrollo, garantizando la escalabilidad, mantenibilidad y calidad técnica del sistema.

## 1. Modelo SDD (Spec Driven Design)
El Desarrollo Orientado a Especificaciones (Spec Driven Design) será la base de nuestro ciclo de vida.
- **Definición Clara**: Todas las funcionalidades deben iniciar con un documento de especificación formal (Spec) antes de escribir código.
- **Alineación**: Asegura que los requerimientos de negocio, arquitectónicos y de usuario estén completamente alineados.
- **Ejecución**: Se utilizará el documento de especificación para derivar las pruebas, la implementación y los criterios de aceptación.

## 2. Arquitecturas Limpias (Clean Architecture)
Para mantener el código mantenible y desacoplado de frameworks y herramientas externas, seguiremos el principio de dependencia de las Arquitecturas Limpias.
- **Regla de Dependencia**: Las dependencias del código fuente solo pueden apuntar hacia adentro, hacia las políticas de alto nivel (entidades y casos de uso).
- **Independencia**: El sistema debe ser independiente de la interfaz de usuario, base de datos y frameworks externos.
- **Capas**:
  - **Entidades**: Reglas de negocio core de la empresa.
  - **Casos de Uso**: Reglas de negocio de la aplicación.
  - **Adaptadores de Interfaz**: Convierten datos de los formatos convenientes para casos de uso/entidades a formatos para herramientas externas (Ej. Controladores, Presentadores).
  - **Frameworks y Drivers**: Base de datos, Interfaz de usuario, dispositivos externos.

## 3. Arquitectura Hexagonal (Puertos y Adaptadores)
Como implementación específica de la Arquitectura Limpia, utilizaremos el patrón de Arquitectura Hexagonal.
- **Core del Dominio Aislado**: El núcleo de la aplicación no tendrá dependencias externas.
- **Puertos**: Las interfaces a través de las cuales el núcleo interactúa con el mundo exterior (Interfaces de entrada y salida).
- **Adaptadores**: Implementaciones concretas de los puertos (Ej: Adaptador REST para entrada, Adaptador JPA/SQL para salida).
- **Beneficios**: Facilita enormemente las pruebas unitarias (mediante mocks de los puertos de salida) y permite cambiar infraestructuras sin tocar el código de negocio, mejorando notablemente el rendimiento durante automatización de testing.

## 4. Plan SCM (Software Configuration Management)
El plan de Gestión de la Configuración del Sistema asegura un control riguroso sobre las versiones, cambios y despliegues del código base.
- **Control de Versiones**: Uso estricto de repositorios (ej. Git).
- **Estrategia de Ramas (Branching)**:
  - `main` / `master`: Contiene únicamente código estable y listo para producción.
  - `develop`: Rama de integración para el próximo release.
  - `feature/*`: Ramas para el desarrollo de nuevas especificaciones (basadas en `develop`).
  - `bugfix/*`: Ramas para correcciones de errores identificados en desarrollo o testing.
  - `release/*`: Ramas destinadas a la preparación de nuevas versiones para producción.
  - `hotfix/*`: Para correcciones urgentes en producción.
- **Trazabilidad del Código**: Los commits y las ramas deben vincularse directamente con las aprobaciones SDD.
- **Integración Continua (CI)**: Todo PR (Pull Request) requiere pasar pruebas automáticas, linters y revisión por pares antes del merge.

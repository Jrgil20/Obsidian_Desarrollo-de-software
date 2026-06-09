# 📝 Preparación: Segundo Examen Parcial (30%)

> [!summary] detalles del Examen
> - **Fecha original:** Jueves 04 de Junio
> - **Estatus:** `[Completado / En Repaso]`
> - **Core conceptual:** Arquitectura de Software, Domain-Driven Design (DDD) y Estrategias de Testing Avanzado.

---

## 🧭 Índice de Clases y Contenido Temático

### 📂 Bloque 1: Arquitectura de Software (08 de Mayo)
- **Conceptos Clave:**
	- Capas y su semántica clásica.
	- Arquitectura de Puertos y Adaptadores.
	- Arquitectura Hexagonal.
	- *Clean Architecture* y la **Regla de Dependencia** (las dependencias de código solo deben apuntar hacia adentro, hacia las reglas de negocio).
- **📚 Material de Referencia Local:**
	- [[Larman, Craig - Applying UML and patterns_ an introduction to object-oriented analysis and design and the unified proces (2016, Prentice Hall) - libgen.li.pdf | Larman, Craig - Applying UML and patterns]]
	- ![[Clean Architecture with .NET - Dino Esposito.pdf#page=1]] (Capítulos 1-3)
	- ![[Adaptive Code - Gary McLean Hall.pdf]] (Capítulo 3)
	- ![[Clean Architecture - Robert Martin.pdf]] (Capítulos 15, 17, 18, 22, 24, 25, 28)
	- ![[Clean Mobile Architecture - Petros Efthymiou.pdf]] (Capítulos 14-16, 19)

### 📂 Bloque 2: Domain-Driven Design - DDD (11 al 18 de Mayo)
- **11 de Mayo (Diseño Estratégico):**
	- Relación entre Arquitectura de software y DDD.
	- Espacio del problema vs. Espacio de la solución.
	- Patrones estratégicos: *Bounded Contexts*, *Ubiquitous Language* y *Context Mapping*.
- **15 de Mayo (Diseño Táctico):**
	- Modelado del Modelo de Dominio rico.
	- Patrones tácticos fundamentales:
		- `Value Objects` (Inmutabilidad y equidad por valor).
		- `Entidades` (Identidad única a lo largo del tiempo).
		- `Servicios de Dominio` (Operaciones de negocio sin estado que no pertenecen a una entidad).
		- `Eventos de Dominio` (Notificación de sucesos relevantes para el negocio).
		- `Agregados` (Clúster de objetos asociados con una raíz de agregado que mantiene los invariantes).
		- `Fábricas` y `Repositorios` (Ciclo de vida del agregado).
	- **Recurso adjunto:** 📄 ![[Clase DDD Patrones Tacticos.html]]
- **18 de Mayo (Modelado Práctico):**
	- Ejercicios aplicados de modelado de la capa de dominio.
- **📚 Material de Referencia Local:**
	- ![[Patterns, Principles, and Practices of Domain-Driven Design - Millett.pdf]] (Capítulos 15-21)
	- ![[Hands-on Domain-driven Design by example - Plöd.pdf]]

### 📂 Bloque 3: Estrategias de Testing (22 al 29 de Mayo)
- **22 de Mayo (Introducción y Metodologías):**
	- Evolución y taxonomía del Testing.
	- Enfoques guiados: *Test-Driven Development* (TDD), *Behavior-Driven Development* (BDD) y *Spec-driven development*.
- **25 de Mayo (Unit Testing Puro):**
	- Anatomía de una prueba unitaria robusta.
	- Problema de los tests frágiles y cómo asegurar la mantenibilidad a largo plazo.
	- Los 4 pilares de un buen Unit Test.
	- Patrones de *Test Doubles*: Mocks, Stubs, Spies, Fakes y Dummies.
- **29 de Mayo (Patrones Avanzados en Testing):**
	- Patrón **AAA** (*Arrange, Act, Assert*).
	- Patrón **Object Mother** para la generación de datos de prueba fijos.
	- Uso del patrón **Builder** enfocado en Testing para inicializar estados complejos de forma fluida.
	- Concepto de *Structural coupling* a través de la API de pruebas.
- **📚 Material de Referencia Local:**
	- ![[Unit Testing Principles, Practices, and Patterns - Khorikov.pdf]] (Capítulos 2-5)
	- ![[Effective Software Testing - Aniche.pdf]] (Capítulos 4, 7, 10)

---
## 🗂️ Banco de Práctica (Modelos de Examen)

> [!todo] Simulacros de Examen
> Espacios de trabajo dedicados para cada examen pasado, estructurados como repositorios independientes para mantener la bóveda limpia.

- 📂 **[[Parcial_28112025|Examen Parcial - 28 de Noviembre de 2025]]** -> *DDD & Testing Avanzado (Incluye Guía Oficial).*
- 📂 **[[Parcial_19122024|Examen Parcial - 19 de Diciembre de 2024]]** -> *Clean & Hexagonal Architecture.*
- 📂 **[[Parcial_31052024|Examen Parcial - 31 de Mayo de 2024]]** -> *Refactoring, SOLID & Patrones.*
---

## 🤖 Insights de IA (NotebookLM & Copilot)

> [!note] Notas de Soporte y Prompts
> - **Enlace de Contexto:** Puedes saltar directo a tu **[NotebookLM de Diseño](https://notebooklm.google.com/notebook/b526e8f9-b7bd-43ff-bb52-6d36136870cb?authuser=1)** para interrogar al asistente sobre las diferencias exactas entre un *Repository* de DDD y un *Data Access Object (DAO)* tradicional.
> - **Prompt Útil para Práctica:** >   ```text
>   Actúa como un profesor experto en arquitectura de software de la UCAB. Evaluando Clean Architecture y DDD, analiza mi solución al Modelo de Examen A y busca violaciones a la Regla de Dependencia o fallas en el encapsulamiento del Agregado.
>   ```
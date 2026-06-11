# 📦 Workspace: Caso Práctico Biblioteca Virtual "DuBook"

> [!abstract] Índices del Repositorio
> Espacio de trabajo modular para el diseño del modelo de dominio de la plataforma DuBook. El diseño aborda la consistencia transaccional en la asignación de ejemplares finitos y el desacoplamiento algorítmico del motor de recomendaciones personalizadas.

---

## 🏛️ Diseño del Modelo de Dominio (Domain-Driven Design)

### 1. Entidades y Objetos de Dominio Identificados

| Concepto de Dominio | Clasificación DDD | Atributos Críticos | Responsabilidad e Invariantes |
| :--- | :--- | :--- | :--- |
| **`Lector`** | Raíz de Agregado | `LectorId`, `Status` (Enum), `CurrentLoansCount` (int) | Guardián del estado del usuario. Valida de forma atómica si el usuario está `ACTIVO` o `INHABILITADO` y si ha alcanzado el límite máximo de préstamos simultáneos. |
| **`Libro`** | Raíz de Agregado / Entidad | `LibroId`, `LoanPolicy` (Value Object), `Stock` (Value Object) | Controla las reglas de circulación del título. Determina si el libro permite exclusivamente lectura diurna o préstamos multidía. |
| **`Prestamo`** | Entidad / Agregado | `LoanId`, `LectorId` (Ref), `LibroId` (Ref), `Period` (DateRange) | Registra el hecho transaccional del préstamo. Realiza el seguimiento de la fecha de entrega y la fecha límite de devolución. |
| **`LoanPolicy`** | Value Object | `IsRestrictedToSameDay: boolean` | Determina de forma inmutable la política de préstamo aplicada a la ficha del libro. |
| **`BookStock`** | Value Object | `AvailableCopies: int` | Controla la finitud de los ejemplares físicos indexados en la biblioteca virtual. Lanza una excepción si se intenta decrementar estando en cero. |
| **`UserPreference`** | Value Object | `Keywords: List<string>`, `ReadingHistoryHash` | Encapsula el vector numérico o atributos de gustos del usuario necesarios para alimentar las recomendaciones. |

---

### 2. Servicios de Dominio (Pureza de Procesos Stateless)

* **`ServicioAutorizacionPrestamo` (BorrowingAuthorizationService):**
    * *Responsabilidad:* Validar de forma integral si un préstamo es legal y viable antes de persistirlo en el sistema.
    * *Justificación de Dominio:* La lógica de negocio requiere evaluar tres agregados e invariantes independientes de forma simultánea: verificar que el `Lector` no esté inhabilitado y tenga cupo, comprobar que el `Libro` disponga de copias en su `BookStock`, y validar que el tipo de préstamo se ajuste a la `LoanPolicy`. Como un agregado no debe conocer el estado interno de otro para no romper los límites de consistencia transaccional, este proceso se modela como un servicio puro y *stateless*.

* **`MotorRecomendacionLibros` (BookRecommendationEngine):**
    * *Responsabilidad:* Encapsular el proceso de negocio de la analítica de personalización de contenido, abstrayendo la sobrecarga de información mediante el filtrado de catálogos.
    * *Justificación de Dominio:* Computar recomendaciones no es una responsabilidad biológica ni del `Lector` ni del `Libro`. Requiere cruzar las preferencias cruzadas de múltiples lectores y catálogos de libros. Al no poseer un estado propio que requiera identidad, se centraliza como un Servicio de Dominio que coordina los algoritmos de filtrado.

---

## 🎨 Patrón de Diseño para el Sistema de Recomendaciones

### Patrón Seleccionado: Strategy (Estrategia)

* **Justificación Arquitectónica:** El enunciado del problema detalla explícitamente tres enfoques algorítmicos distintos e intercambiables para resolver el mismo problema de negocio (Filtrado Colaborativo, Filtrado Basado en Contenido y Filtrado Híbrido). El patrón **Strategy** permite aislar estos algoritmos volátiles en sus propias clases concretas bajo una interfaz común (`IRecommendationStrategy`). Esto le permite al `BookRecommendationEngine` (Contexto) cambiar la estrategia en tiempo de ejecución de manera transparente (Open-Closed Principle), dependiendo de factores como el perfil del usuario o la disponibilidad de datos históricos, sin impactar al código cliente.

---

## 💻 Especificación de Firmas y Contratos Técnicos

```csharp
// Definición del Contrato de la Estrategia (Puerto de Algoritmos)
public interface IRecommendationStrategy
{
    // Recibe el contexto del lector y retorna los IDs de los libros sugeridos
    List<string> RecommendBooks(string lectorId, int maxResults);
}

// Implementaciones Concretas e Intercambiables
public class CollaborativeFilteringStrategy : IRecommendationStrategy
{
    public List<string> RecommendBooks(string lectorId, int maxResults)
    {
        // Lógica para identificar usuarios con gustos similares (Vecinos cercanos)
        return new List<string>();
    }
}

public class ContentBasedFilteringStrategy : IRecommendationStrategy
{
    public List<string> RecommendBooks(string lectorId, int maxResults)
    {
        // Lógica basada en atributos del libro y del historial pasado del lector
        return new List<string>();
    }
}

// Contexto del Servicio de Dominio que consume la estrategia
public class BookRecommendationEngine
{
    private IRecommendationStrategy _strategy;

    // Inyección dinámica de la estrategia de filtrado
    public BookRecommendationEngine(IRecommendationStrategy strategy)
    {
        _strategy = strategy;
    }

    public void ChangeStrategy(IRecommendationStrategy newStrategy)
    {
        _strategy = newStrategy;
    }

    public List<string> GetPersonalizedContent(string lectorId, int limit)
    {
        return _strategy.RecommendBooks(lectorId, limit);
    }
}
````

## 📐 Lienzo de Diseño Táctico (Para Resolver en Simulacro)

### Ejercicio 1: Flujo de Ejecución para la Autorización de Préstamos

_Diseña el pseudo-código del Servicio de Dominio que cruza las invariantes antes de emitir un préstamo con éxito:_

C#

```
public class BorrowingAuthorizationService
{
    // Tip UCAB: Recuerda inyectar los repositorios necesarios como puertos
    public Either<string, Prestamo> Authorize(string lectorId, string libroId)
    {
        // 1. Recuperar agregados de los repositorios
        // 2. Validar lector.CanBorrow()
        // 3. Validar libro.HasAvailableStock()
        // 4. Retornar Left(Error) o Right(Prestamo)
        return null; 
    }
}
```

### Ejercicio 2: Diagrama de Texto del Patrón Strategy en DuBook

Plaintext

```
  ┌─────────────────────────────┐
  │  BookRecommendationEngine   │ ───► Usa un contrato abstracto
  └─────────────────────────────┘
                 │
                 ▼
    «interface» IRecommendationStrategy 
                 │
  ┌──────────────┼────────────────────────┐
  ▼              ▼                        ▼
┌──────────┐   ┌─────────────┐   ┌────────────────┐
│ Colaborat│   │ContentBased │   │ HybridStrategy │
└──────────┘   └─────────────┘   └────────────────┘
```

***

### 💡 Consejo del ingeniero para el examen:
Ten mucho cuidado con la entidad **`Libro`** y la gestión de sus ejemplares. En exámenes de este tipo en la UCAB, un error común es modelar las "copias de un libro" como una colección de sub-entidades con ID propio dentro del libro. 

A menos que el enunciado te exija hacerle seguimiento físico individual a cada copia (como saber si la copia física #3 tiene una página rota), debes tratar el número de ejemplares disponibles como un simple **Value Object inmutable (`BookStock`)** con un entero adentro. Esto simplifica drásticamente el control de la concurrencia y optimiza el rendimiento del modelo, demostrando un criterio de diseño limpio y pragmático.


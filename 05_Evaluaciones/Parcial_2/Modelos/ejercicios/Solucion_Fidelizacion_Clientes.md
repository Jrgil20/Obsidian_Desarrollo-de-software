
# 📦 Workspace: Caso Práctico Sistema de Compras y Fidelización de Clientes

> [!abstract] Índices del Repositorio
> Espacio de trabajo modular para el diseño del modelo táctico de DDD enfocado en la actualización dinámica de niveles de fidelidad, algoritmos volátiles y el aislamiento de un sistema de consultas históricas externo de alto desempeño.

---

## 🏛️ Diseño del Modelo Táctico (Domain-Driven Design)

### 1. Value Objects Enriquecidos (Rich Value Objects)

* **`Money` (Monto):** Encapsula el valor numérico (`decimal`) y el tipo de moneda.
    * *Invariantes:* El constructor valida de forma atómica que el monto no sea inferior a cero.
* **`Email` / `PhoneNumber`:** Evitan el antipatrón de *Obsesión Primitiva*. Validan sus respectivas expresiones regulares y estructuras formales directamente en el constructor.
* **`LoyaltyLevel` (Nivel de Fidelidad con Comportamiento):** Representa la jerarquía de estados (`INITIAL`, `SILVER`, `GOLD`, `PLATINUM`, `PREMIUM_PLUS`).
    * *Responsabilidades:* Define las transiciones legales y provee métodos de comparación semántica (ej. `IsEligibleForBonus()`) para evitar que la capa de aplicación use estructuras `if-else` o comparaciones de strings sueltos.

---

### 2. Límites de Agregados y Entidades

#### 📦 Agregado: Customer (Cliente)
* **Raíz del Agregado (Aggregate Root):** `Customer`
    * *Atributos:* `CustomerId` (Identity), `Name`, `Email`, `PhoneNumbers` (Colección de VOs), `CurrentLoyaltyLevel` (LoyaltyLevel).
    * *Invariantes Globales de Consistencia:*
        1. Es el único guardián de su nivel de fidelidad. Su estado solo muta a través de un método de negocio explícito (ej. `customer.UpdateLoyaltyStatus(newLevel)`).
        2. No expone mutadores públicos (*setters*) para asegurar que ningún componente externo modifique arbitrariamente su nivel de fidelidad.

#### 📦 Agregado: Order (Orden de Compra)
* **Raíz del Agregado (Aggregate Root):** `Order`
    * *Atributos:* `OrderId` (Identity), `TotalAmount` (Money), `PlacementDate` (DateTime), `OrderStatus` (PENDING, SUCCESSFUL, FAILED).
    * *Invariantes Globales de Consistencia:*
        1. Una orden en estado `SUCCESSFUL` es un hecho histórico inmutable.
        2. **Separación de Incumbencias (SoC):** Para cumplir estrictamente con el principio de responsabilidad única (SRP), la orden no calcula puntos ni conoce la existencia del cliente; simplemente publica un Evento de Dominio (`PurchaseOrderCompletedEvent`) cuando su estado pasa a exitoso.

---

### 3. Servicios de Dominio (Pureza de Procesos Algorítmicos)

* **`CalculadorPuntosFidelidad` (LoyaltyPointCalculator):**
    * *Responsabilidad:* Alojar el algoritmo complejo e inestable en el tiempo que determina cuántos puntos o qué nivel le corresponde al cliente.
    * *Justificación de Dominio:* El cálculo requiere procesar tres factores concurrentes: el `CurrentLoyaltyLevel` del cliente, el `TotalAmount` de la nueva orden y el **historial completo de compras**. Alojar esta lógica en `Customer` rompería su alta cohesión (el cliente no debe saber computar algoritmos de analítica histórica), y alojarla en `Order` acoplaría la orden al perfil del usuario. Al ser un proceso puro y *stateless*, se modela como un Servicio de Dominio.
    * *Aislamiento del Sistema Externo:* Para interactuar con el sistema de alto desempeño sin contaminar el dominio, el servicio consume un **Puerto (Interface)** denominado `IPurchaseHistoryProvider` (que actúa como una Capa Anticorrupción / ACL). La infraestructura implementará este puerto para realizar las consultas ultra-rápidas.

---

## 🛠️ Plan de Refactorización (Target Architecture Matrix)

| Requisito / Proceso | Patrón DDD / Clean Arq. | Capa de Destino |
| :--- | :--- | :--- |
| Validación de montos y correos | Invariantes en VOs (`Money`, `Email`) | **Dominio** |
| Algoritmo Complejo de Puntos | Servicio de Dominio (`LoyaltyPointCalculator`) | **Dominio** |
| Consulta al sistema de alto desempeño | Puerto (`IPurchaseHistoryProvider`) | **Aplicación (Interface) / Dominio** |
| Implementación técnica de la consulta rápida | Adaptador de Infraestructura (Gateway/ACL) | **Infraestructura** |
| Reacción al éxito de la compra | Evento de Dominio (`PurchaseOrderCompletedEvent`) | **Dominio / Aplicación** |

---

## 📐 Lienzo de Diseño Táctico (Para Resolver en Simulacro)

### Ejercicio 1: Definición del Puerto de Infraestructura (Capa Anticorrupción)
*Diseña el contrato de la interfaz que abstrae al sistema de alto desempeño para que el dominio no sepa qué base de datos o tecnología usa el sistema externo:*

```csharp
// Escribe aquí la interfaz en el lenguaje de tu preferencia (ej. C# / TypeScript)
public interface IPurchaseHistoryProvider 
{
    // Tip: Debe recibir el CustomerId y retornar un DTO de Dominio o una colección de montos/fechas pasadas.
}
````

### Ejercicio 2: Flujo de Ejecución e Interacción de Componentes

_Completa los pasos del Caso de Uso/Servicio de Aplicación que orquesta la actualización cuando se confirma el éxito de la compra:_

Plaintext

```
[Caso de Uso: ProcessSuccessfulOrder]
  ├── 1. Recibir el ID de la Orden Exitosa
  ├── 2. Invocar el Puerto para traer el historial: __________________________________
  ├── 3. Pasar el historial, nivel actual y monto nuevo al Servicio de Dominio: _______
  ├── 4. Customer AR: Ejecuta la mutación de estado con el resultado del cálculo
  └── 5. Unidad de Trabajo / Repository: Persiste el nuevo estado del Cliente
```

***

### 💡 Consejo del ingeniero para el examen:
Ojo con el detalle del enunciado: *"por razones de eficiencia el histórico de las órdenes se maneja en un sistema aparte"*. En la rúbrica de la UCAB, esto te está gritando que **no puedes hacer un Join SQL tradicional** ni usar el repositorio estándar del agregado `Order` para traer el historial. 

La existencia de un sistema externo de alto desempeño exige obligatoriamente la creación de un puerto dedicado (`IPurchaseHistoryProvider` o `IPurchaseHistoryGateway`). Al hacerlo así, demuestras que sabes aplicar una **Capa Anticorrupción (ACL)** para proteger tu modelo de dominio de las decisiones de optimización física de la infraestructura.

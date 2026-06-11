# 🔍 Ejercicio de Refactorización: Árbol de Validaciones Complejas (Tree of Responsibility)

> [!info] Variación de Patrón Gof: Tree of Responsibility + Either Monad
> Diseño de un motor de validación extensible para órdenes de venta corporativas. A diferencia de la Cadena de Responsabilidad clásica, esta variante estructural bifurca la ejecución en múltiples manejadores paralelos/hijos (*composite/tree structure*) y acumula de forma exhaustiva todos los fallos del sistema sin interrumpir el flujo ante el primer error, utilizando tipos de retorno `Either<L, R>`.

---

## 📐 Diseño de Entidades y Objetos de Valor (Sales Order Aggregate)

Para evitar el antipatrón de *Obsesión Primitiva* y asegurar que el Árbol de Validación reciba tipos fuertemente acoplados a las reglas de negocio de la UCAB, el agregado de la Orden se compone estructuralmente de la siguiente manera:

* **`CustomerInfo` (Entidad/VO):** Agrupa `Status` (Enum: ACTIVO, INACTIVO), `Address` (VO), `Telephone` (VO) y `CreditProfile` (VO).
* **`Address` (Value Object):** Contiene los campos mandatorios `StreetAddress`, `PostalCode` y `Country`.
* **`Telephone` (Value Object):** Contiene `OperatorPrefix` (string de 3 dígitos) y `LineNumber` (string de 7 dígitos).
* **`CreditProfile` (Value Object):** Contiene `CurrentBalance` (Money) y `CreditLimit` (Money).
* **`OrderItem` (Entidad):** Contiene `ProductId`, `Quantity` (int), `UnitPrice` (Money) y `ListPriceReference` (Money).
* **`SalesOrder` (Raíz del Agregado):** Contiene `OrderId`, `Customer` (CustomerInfo), la lista indexada `Items` (Collection de OrderItem) y el método computado `CalculateTotalVenta(): Money`.

---

## 🏛️ Arquitectura del Árbol de Responsabilidad (Tree of Responsibility)

### Abstracción Base del Validador

Cada nodo del árbol actúa simultáneamente como un procesador local y como un despachador de múltiples sub-reglas (*Composite Pattern Behavior*). 

La abstracción define el contrato estricto de acumulación de errores mediante la firma genérica externa `Either<List<ValidationError>, SalesOrder>`, donde:
* **`Left (L)`**: Contiene la colección `List<ValidationError>` con todos los errores acumulados en ese nodo y sus sub-árboles hijos.
* **`Right (R)`**: Contiene la entidad `SalesOrder` intacta, indicando que el sub-árbol completo pasó las validaciones de negocio de forma exitosa.

---

## 💻 Especificación de Firmas y Contratos Técnicos

### 1. Clases Base y Tipos de Soporte

```csharp
public class ValidationError 
{
    public string RuleName { get; }
    public string ErrorMessage { get; }

    public ValidationError(string ruleName, string errorMessage)
    {
        RuleName = ruleName;
        ErrorMessage = errorMessage;
    }
}

public abstract class AbstractOrderValidator
{
    // Permite la ramificación del árbol al soportar múltiples sucesores en paralelo
    protected List<AbstractOrderValidator> ChildValidators = new List<AbstractOrderValidator>();

    public void AddChild(AbstractOrderValidator validator)
    {
        this.ChildValidators.Add(validator);
    }

    // Método plantilla (Template Method) que orquesta la ejecución local y la de sus hijos
    public Either<List<ValidationError>, SalesOrder> Validate(SalesOrder order)
    {
        var errors = new List<ValidationError>();

        // 1. Ejecución de la regla local del nodo
        var localResult = this.EvaluateLocalRule(order);
        if (localResult.IsLeft)
        {
            errors.AddRange(localResult.GetLeft());
        }

        // 2. Recorrer de forma recursiva todo el sub-árbol de sucesores (Comportamiento exhaustivo)
        foreach (var child in ChildValidators)
        {
            var childResult = child.Validate(order);
            if (childResult.IsLeft)
            {
                errors.AddRange(childResult.GetLeft());
            }
        }

        // 3. Evaluar el acumulador transaccional del sub-árbol
        if (errors.Count > 0)
        {
            return Either<List<ValidationError>, SalesOrder>.Left(errors);
        }

        return Either<List<ValidationError>, SalesOrder>.Right(order);
    }

    // Contrato obligatorio para cada Handler concreto
    protected abstract Either<List<ValidationError>, SalesOrder> EvaluateLocalRule(SalesOrder order);
}
````

### 2. Implementaciones Concretas de las Reglas Mandatorias (Handlers)

#### A. `CustomerStatusValidator`

- **Responsabilidad:** Validar el estado operacional del cliente.
    
- **Firma:** `protected override Either<List<ValidationError>, SalesOrder> EvaluateLocalRule(SalesOrder order)`
    
- **Lógica:** Si `order.Customer.Status != CustomerStatus.ACTIVO`, retorna `Left` con un `ValidationError("CustomerStatus", "El cliente debe tener el estatus de Activo")`.
    

#### B. `CustomerAddressValidator`

- **Responsabilidad:** Validar la integridad física del despacho del cliente.
    
- **Firma:** `protected override Either<List<ValidationError>, SalesOrder> EvaluateLocalRule(SalesOrder order)`
    
- **Lógica:** Evalúa si `StreetAddress`, `PostalCode` o `Country` en `order.Customer.Address` son nulos, vacíos o espacios en blanco. Retorna `Left` con la lista de campos faltantes si aplica.
    

#### C. `CustomerTelephoneValidator`

- **Responsabilidad:** Validar la máscara reglamentaria de telecomunicaciones.
    
- **Firma:** `protected override Either<List<ValidationError>, SalesOrder> EvaluateLocalRule(SalesOrder order)`
    
- **Lógica:** Verifica mediante expresiones regulares o longitud de cadena que `OperatorPrefix` tenga exactamente 3 dígitos numéricos y `LineNumber` exactamente 7 dígitos numéricos.
    

#### D. `CustomerCreditLineValidator`

- **Responsabilidad:** Control de riesgo financiero pre-despacho.
    
- **Firma:** `protected override Either<List<ValidationError>, SalesOrder> EvaluateLocalRule(SalesOrder order)`
    
- **Lógica:** Calcula `var totalVenta = order.CalculateTotalVenta()`. Evalúa si `(totalVenta + order.Customer.CreditProfile.CurrentBalance) > order.Customer.CreditProfile.CreditLimit`. Si se excede el límite, emite un fallo de riesgo financiero en un `Left`.
    

#### E. `OrderItemsValidator`

- **Responsabilidad:** Validar los invariantes de rentabilidad y consistencia física de las líneas de la orden.
    
- **Firma:** `protected override Either<List<ValidationError>, SalesOrder> EvaluateLocalRule(SalesOrder order)`
    
- **Lógica:** Recorre la colección de ítems. Valida que `Quantity > 0`. Computa si el `UnitPrice` es inferior al `ListPriceReference * 0.80` (violación del umbral de descuento permitido del 20%). Acumula todas las líneas infractoras en el `Left`.
    

## 🛠️ Matriz del Target Architecture (Open-Closed Principle)

|**Componente del Sistema**|**Patrón / Responsabilidad**|**Beneficio en el Ecosistema del CRM**|
|---|---|---|
|`AbstractOrderValidator`|Componente Base / Composite|Permite agregar validaciones futuras (fraude, aduana, impuestos) sin modificar los controladores.|
|`Either<List<Error>, Order>`|Mónada de Flujo Funcional|Elimina el uso de excepciones (`try-catch`) para el control de flujo de reglas comerciales, haciéndolo predecible.|
|`Composite Execution Loop`|Recorrido Exhaustivo|Satisface el requerimiento de retornar todos los errores simultáneamente a la interfaz gráfica en lugar de forzar al usuario a enviar el formulario múltiples veces.|

## 📐 Lienzo de Diseño Táctico (Para Resolver en Simulacro)

### Ejercicio 1: Configuración de la Raíz de Composición del Árbol (Composition Root)

_Escribe el pseudo-código que construye la topología del árbol en el servicio de aplicación vinculando los nodos hijos:_

C#

```
public class CreateSalesOrderUseCase
{
    public Either<List<ValidationError>, SalesOrder> Execute(SalesOrder order)
    {
        // 1. Instanciar la raíz del árbol (ej. un validador maestro o el primer validador)
        var rootValidator = new CustomerStatusValidator();

        // 2. Instanciar los nodos hijos
        var addressVal = new CustomerAddressValidator();
        var phoneVal = new CustomerTelephoneValidator();
        var creditVal = new CustomerCreditLineValidator();
        var itemsVal = new OrderItemsValidator();

        // 3. Modelar la estructura del ARBOL (Bifurcaciones)
        // [Completa el armado del árbol usando rootValidator.AddChild(...) aquí]

        // 4. Ejecutar y retornar la evaluación atómica
        return rootValidator.Validate(order);
    }
}
```

### Ejercicio 2: Flujo de Evaluación de Errores Exhaustivos (Diagrama de Texto)

Plaintext

```
[SalesOrder Client Request] 
      │
      ▼
┌─────────────────────────────────┐
│     CustomerStatusValidator     │ ──(Falla Local)──> [Agrega Error 1]
└─────────────────────────────────┘
      │
      ├─► ┌─────────────────────────────┐
      │   │  CustomerAddressValidator   │ ──(Exitoso)──────> [No aporta errores]
      │   └─────────────────────────────┘
      │
      └─► ┌─────────────────────────────┐
          │     OrderItemsValidator     │ ──(Falla Local)──> [Agrega Error 2]
          └─────────────────────────────┘
      │
      ▼
 [Resultado Final: Either.Left] ───> Retorna List { Error 1, Error 2 } al controlador HTTP
```

***

### 💡 Consejo del ingeniero para el examen:
Ojo con el método plantilla (`Template Method`) que diseñé en la clase abstracta `AbstractOrderValidator`. En los exámenes de ingeniería informática de la UCAB, cuando te piden modificar un patrón para que sea **exhaustivo** (que guarde todos los errores en vez de romper la ejecución de inmediato), la clave técnica está en **no colocar sentencias `return` prematuras** dentro del bucle que recorre los elementos hijos. 

Fíjate cómo el bucle `foreach` sigue ejecutando el método `.Validate(order)` en cada componente del árbol sin importar cuántos `Left` se vayan encontrando en el camino, y solo al final de todo el recorrido estructural se evalúa si la lista de errores contiene elementos para decidir si se retorna un `Left` o un `Right`. Esto blinda completamente la nota frente a la rúbrica de evaluación.

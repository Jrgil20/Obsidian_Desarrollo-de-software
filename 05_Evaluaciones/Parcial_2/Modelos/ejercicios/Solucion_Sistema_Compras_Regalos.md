# 🔍 Ejercicio de Refactorización: Sistema de Compras y Regalos (Hexagonal + DDD)

> [!danger] Code Smell Clínico: God Service & Modelo Anémico
> El servicio original `ComprarORegalarProductoXCantidad` opera como un "Script de Transacción" que viola flagrantemente SRP, DIP, SoC y OCP. Mezcla la obtención de datos, políticas de descuento, cálculo de impuestos de aduana, control transaccional de saldos, persistencia en archivos de log y control de flujo mediante excepciones.

---

## 🏛️ 1. Arquitectura Hexagonal y Límites de Capas (Inside/Outside)

El sistema se desacopla aislando la lógica de negocio pura de la infraestructura mediante **Puertos** (Interfaces en el Core) y **Adaptadores** (Implementaciones externas):

```text
  [Outside: Infraestructura] ──► (Inyecta) ──► [Puertos / Interfaces] ──► [Inside: Core]
  - SqlUserRepository                          - IUserRepository          - ProcessPurchaseUseCase
  - AccountingTaxAdapter                       - ITaxService              - PurchasePricingService
  - FileLoggerAspect                           - IEmailService            - User / Product AR
````

### Puertos de Entrada (Driving Ports)

- `IProcessPurchaseUseCase`: Contrato consumido por la capa de presentación (API REST / CLI) para disparar la compra.
    

### Puertos de Salida (Driven Ports)

- `IUserRepository`: Persistencia y recuperación del agregado `User`.
    
- `IProductRepository`: Persistencia y recuperación del agregado `Product`.
    
- `ITaxService`: Gateway de comunicación con el sistema contable externo.
    
- `IEmailService`: Abstracción para el envío de comprobantes digitales.
    

## 🧩 2. Modelo Táctico DDD (El Corazón de la Lógica)

### Value Objects Enriquecidos (Inmutables)

- **`Money`:** Encapsula operaciones aritméticas seguras con punto decimal fijo (`decimal`).
    
    - _Invariantes:_ Impide la creación de montos negativos.
        
    - _Comportamiento:_ Provee métodos semánticos como `Subtract(Money)`, `Add(Money)`, `Multiply(int cantidad)`.
        
- **`Quantity`:** Tipo micro-objeto para el manejo de volúmenes físicos.
    
    - _Invariantes:_ Debe ser estrictamente mayor a cero ($> 0$).
        
- **`Discount`:** Modificador inmutable que representa porcentajes o montos fijos a deducir.
    

### Raíces de Agregados (Aggregate Roots Ricos)

- **`Product` (AR):**
    
    - _Atributos:_ `ProductId`, `BasePrice` (`Money`), `IsOnPromotion` (`boolean`), `PromotionDiscount` (`Discount`).
        
- **`User` (AR):**
    
    - _Atributos:_ `UserId`, `BirthMonth` (`int`), `Balance` (`Money`).
        
    - _Invariantes de Consistencia Transaccional:_ * **`Charge(Money amount)`**: En lugar de verificar el saldo por fuera, el agregado expone este método. Si `Balance < amount`, el objeto rechaza la mutación interna y retorna un fallo de negocio estructurado. Evita el modelo anémico.
        

## ⚙️ 3. Servicios de Dominio y Patrones de Diseño Aplicados

### A. Política de Precios Acumulables: Patrón Strategy

Para resolver de forma limpia la regla de negocio _(d.i.D) "Se pueden aplicar los dos descuentos al mismo tiempo"_, se implementa una cadena o colección de estrategias de descuento que cumplen con el Principio de Abierto/Cerrado (OCP).

- **`IDiscountStrategy` (Interfaz de Dominio):**
    
    - Firma: `Money ApplyDiscount(Product product, User user, Money currentPrice);`
        
- **`BirthdayDiscountStrategy`:** Aplica un $10\%$ de descuento si `user.BirthMonth == DateTime.Now.Month`.
    
- **`PromotionDiscountStrategy`:** Aplica la deducción de la promoción si `product.IsOnPromotion` es verdadero.
    

### B. Servicio de Dominio: `PurchasePricingService` (Stateless)

- **Responsabilidad:** Calcular de forma pura el costo transaccional final bruto y neto de la operación.
    
- **Justificación:** Involucra la evaluación cruzada entre `User`, `Product`, las estrategias de descuento y el consumo condicional de impuestos externos.
    
- **Lógica Interna:**
    
    1. Computa el precio base unitario aplicando la lista de `IEnumerable<IDiscountStrategy>`.
        
    2. Multiplica el precio resultante por la `Quantity`.
        
    3. Si `esRegalo` es falso, consume el puerto `ITaxService` para agregar los impuestos del sistema contable; si es verdadero, omite los impuestos tratándolo como donación inmutable.
        

## 🧪 4. Programación Genérica y AOP (Separación de Preocupaciones)

### Control de Flujo Funcional con `Result<T>`

Para erradicar el uso del lanzamiento de excepciones técnicas como control de flujo regular de negocio (ej. `UsuarioNoExiste`, `SaldoInsuficiente`), se introduce el tipo genérico estructural `Result<T>`:

C#

```
public class Result<T>
{
    public T Value { get; }
    public List<string> Errors { get; }
    public bool IsSuccess => Errors.Count == 0;
    public bool IsFailure => !IsSuccess;

    private Result(T value) { Value = value; Errors = new List<string>(); }
    private Result(List<string> errors) { Errors = errors; }

    public static Result<T> Success(T value) => new Result<T>(value);
    public static Result<T> Fail(string error) => new Result<T>(new List<string> { error });
}
```

### Manejo de Preocupaciones Transversales (AOP mediante Decoradores)

Toda la lógica de Logging de fallos, auditorías y captura de errores genéricos del sistema de archivos se extrae del Caso de Uso e Infraestructura, encapsulándose en un **Decorator**:

C#

```
public class PurchaseLoggingDecorator : IProcessPurchaseUseCase
{
    private readonly IProcessPurchaseUseCase _decoratee;
    private readonly IFileLogger _logger; // Puerto de logging

    public PurchaseLoggingDecorator(IProcessPurchaseUseCase decoratee, IFileLogger logger)
    {
        _decoratee = decoratee;
        _logger = logger;
    }

    public Result<PurchaseSummary> Execute(PurchaseCommand command)
    {
        try
        {
            var result = _decoratee.Execute(command);
            if (result.IsFailure)
            {
                // Registro centralizado de fallos de negocio (Saldo Insuficiente, Inexistencia)
                _logger.LogWarning($"Operación Fallida para Usuario {command.UserId}: {string.Join(", ", result.Errors)}");
            }
            return result;
        }
        catch (Exception ex)
        {
            // Captura de excepciones catastróficas del sistema
            _logger.LogError($"Falla Crítica del Sistema: {ex.Message}", ex);
            return Result<PurchaseSummary>.Fail("Error técnico interno en el procesamiento.");
        }
    }
}
```

## 💻 5. Flujo de Ejecución Limpio en el Caso de Uso (Orquestador)

El caso de uso `ProcessPurchaseUseCase` es **humilde**: no calcula precios, no sabe dónde se guardan los datos, ni sabe cómo enviar correos. Solo coordina:

C#

```
public class ProcessPurchaseUseCase : IProcessPurchaseUseCase
{
    private readonly IUserRepository _userRepository;
    private readonly IProductRepository _productRepository;
    private readonly PurchasePricingService _pricingService;
    private readonly IEmailService _emailService;

    public Result<PurchaseSummary> Execute(PurchaseCommand command)
    {
        // 1. Recuperación de Agregados Básicos
        var buyer = _userRepository.FindById(command.UserId);
        if (buyer == null) return Result<PurchaseSummary>.Fail("Usuario comprador no existe.");

        var product = _productRepository.FindById(command.ProductId);
        if (product == null) return Result<PurchaseSummary>.Fail("Producto especificado no existe.");

        User beneficiary = null;

        // 2. Validación explícita de la regla de negocio del Beneficiario
        if (command.UserIdBeneficiario != null)
        {
            beneficiary = _userRepository.FindById(command.UserIdBeneficiario);
            if (beneficiary == null && command.EsRegalo)
            {
                // Rompe con flujo Left/Failure sin lanzar excepciones pesadas
                return Result<PurchaseSummary>.Fail("El usuario beneficiario especificado para el regalo no existe.");
            }
        }

        // 3. Delegación del cálculo complejo al Servicio de Dominio
        Money totalCost = _pricingService.CalculateTotalCost(product, buyer, command.Quantity, command.EsRegalo);

        // 4. Mutación transaccional protegida dentro del Agregado Root
        var chargeResult = buyer.Charge(totalCost);
        if (chargeResult.IsFailure) return Result<PurchaseSummary>.Fail(chargeResult.Errors.First());

        // 5. Persistencia del nuevo estado del sistema
        _userRepository.Save(buyer);

        // 6. Despacho de Notificaciones según la política de la Compra / Regalo
        if (command.EsRegalo)
        {
            _emailService.Send(beneficiary.Email, $"¡Has recibido un regalo de {buyer.Name}! Producto: {product.Name}, Cantidad: {command.Quantity}");
        }
        else
        {
            _emailService.Send(buyer.Email, $"Confirmación de tu compra. Total Pagado: {totalCost}. Cantidad: {command.Quantity}");
        }

        return Result<PurchaseSummary>.Success(new PurchaseSummary(buyer.UserId, totalCost));
    }
}
```

## 📐 Lienzo de Diseño Táctico (Para Resolver en Simulacro)

### Ejercicio 1: Topología de las Estrategias de Descuento

_Completa el diseño del bucle que procesa los descuentos acumulativos dentro del `PurchasePricingService`:_

C#

```
public class PurchasePricingService
{
    private readonly List<IDiscountStrategy> _discountStrategies;
    private readonly ITaxService _taxService;

    public Money CalculateTotalCost(Product product, User user, int quantity, bool esRegalo)
    {
        Money unitPrice = product.BasePrice;

        // 1. Aplicar la tubería de descuentos secuenciales (Strategy Pattern Loop)
        foreach (var strategy in _discountStrategies)
        {
            // unitPrice = [Completa llamando a la estrategia actual]
        }

        Money totalBruto = unitPrice.Multiply(quantity);

        // 2. Aplicar lógica de impuestos de aduana / contabilidad condicional
        if (esRegalo) return totalBruto;

        Money taxes = _taxService.CalculateTaxesForAmount(totalBruto);
        return totalBruto.Add(taxes);
    }
}
```

### Ejercicio 2: Diagrama de Pasaje por Capas Hexagonales (Texto)

Plaintext

```
  [HTTP POST: /api/purchases] ──► Consume ──► [LoggingDecorator] ──► Llama ──► [ProcessPurchaseUseCase]
                                                                                      │
         ┌────────────────────────────────────────────────────────────────────────────┤
         ▼                                                                            ▼
 [PurchasePricingService] ──► Evalúa Estrategias                         [User Aggregate Root]
         │                                                                    │
         └──► Consume Puerto ──► [ITaxService]                                └──► Invoca .Charge()
```

***

### 💡 Tip de Parcial para este Ejercicio:
Ojo con el **Manejo del Cumpleaños y las Promociones**. Si pones un condicional rígido dentro del servicio (`if (user.IsBirthday)`), te van a penalizar el examen por violar el Principio de Abierto/Cerrado (OCP). 

Al encapsular cada política de descuento en un objeto que implementa `IDiscountStrategy`, el `PurchasePricingService` se vuelve completamente agnóstico de las reglas comerciales. Si el próximo semestre la empresa decide meter un descuento de "Black Friday", simplemente creas `BlackFridayDiscountStrategy` y la inyectas en la lista del constructor sin tocar una sola línea de código del servicio principal.

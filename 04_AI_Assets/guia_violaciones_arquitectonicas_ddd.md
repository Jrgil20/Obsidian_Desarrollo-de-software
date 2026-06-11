# Guía de Violaciones Arquitectónicas y Refactorización en DDD / Clean Architecture

> **Objetivo:** Detectar, clasificar y corregir violaciones comunes del modelo de dominio y de la regla de dependencia en sistemas que siguen los principios de DDD, Arquitectura Limpia o Hexagonal.

---

## Índice

1. [Marco de Referencia](#1-marco-de-referencia)
2. [Violaciones de Capa de Dominio](#2-violaciones-de-capa-de-dominio)
   - 2.1 [Obsesión Primitiva → Value Object](#21-obsesión-primitiva--value-object)
   - 2.2 [Modelo de Dominio Anémico](#22-modelo-de-dominio-anémico)
   - 2.3 [Invariantes no protegidas](#23-invariantes-no-protegidas)
3. [Violaciones de Agregados](#3-violaciones-de-agregados)
   - 3.1 [Acceso directo a entidades internas](#31-acceso-directo-a-entidades-internas)
   - 3.2 [Aggregate Root sin responsabilidades reales](#32-aggregate-root-sin-responsabilidades-reales)
4. [Violaciones de Capa de Aplicación](#4-violaciones-de-capa-de-aplicación)
   - 4.1 [Lógica de negocio en Application Services](#41-lógica-de-negocio-en-application-services)
   - 4.2 [Fuga de infraestructura hacia Application / Domain](#42-fuga-de-infraestructura-hacia-application--domain)
5. [Violaciones en el Manejo de Servicios Externos](#5-violaciones-en-el-manejo-de-servicios-externos)
   - 5.1 [Excepciones técnicas expuestas al dominio](#51-excepciones-técnicas-expuestas-al-dominio)
   - 5.2 [Ausencia de capa anticorrupción (ACL)](#52-ausencia-de-capa-anticorrupción-acl)
6. [Violaciones de la Regla de Dependencia](#6-violaciones-de-la-regla-de-dependencia)
7. [Tabla de Detección Rápida](#7-tabla-de-detección-rápida)
8. [Checklist de Refactorización](#8-checklist-de-refactorización)

---

## 1. Marco de Referencia

La **Regla de Dependencia** es el principio central que todas las demás violaciones terminan rompiendo:

```
[ Infraestructura ] → [ Aplicación ] → [ Dominio ]
         ↑                  ↑               ↑
    (BD, APIs,          (Orquesta)      (Núcleo puro,
     UI, Libs)                          sin dependencias
                                        externas)
```

> **Regla fundamental:** Las flechas de dependencia apuntan **siempre hacia adentro**. El dominio no conoce la infraestructura. La violación de esta regla es la raíz de la mayoría de los problemas de mantenibilidad.

**Tipos de objetos en el dominio:**

| Tipo              | Identidad | Mutabilidad | Responsabilidad                          |
|-------------------|-----------|-------------|------------------------------------------|
| `Entity`          | Sí (ID)   | Mutable     | Encapsula estado + comportamiento propio |
| `Value Object`    | No        | Inmutable   | Representa un concepto por su valor      |
| `Aggregate Root`  | Sí (ID)   | Mutable     | Fachada y guardián del agregado          |
| `Domain Service`  | No        | Sin estado  | Operaciones multi-entidad o complejas    |

---

## 2. Violaciones de Capa de Dominio

### 2.1 Obsesión Primitiva → Value Object

**Señal de alerta:** Un concepto del negocio con reglas propias (rango, formato, unidad) está representado como `int`, `float`, `string`, etc.

#### ❌ Violación

```typescript
class Order {
  private amount: number;  // ← primitivo sin restricciones
  private currency: string;

  setAmount(value: number) {
    this.amount = value; // ← no valida nada: acepta -500, 0, NaN
  }
}

// En el Application Service (error de diseño):
if (order.getAmount() < 0) throw new Error("Monto inválido"); // lógica de dominio filtrada
```

**Problema:** La validación del monto (que es una regla de negocio) vive fuera de la entidad. Está dispersa en los servicios → duplicación de código → modelo anémico.

#### ✅ Refactorización: Extraer Value Object

```typescript
// Value Object: Money
class Money {
  private readonly value: number;
  private readonly currency: string;

  constructor(value: number, currency: string) {
    if (value < 0)        throw new DomainError("El monto no puede ser negativo.");
    if (value === 0)      throw new DomainError("El monto no puede ser cero.");
    if (!currency.match(/^[A-Z]{3}$/)) throw new DomainError("Moneda inválida (ISO 4217).");

    this.value = value;
    this.currency = currency;
  }

  add(other: Money): Money {
    if (this.currency !== other.currency)
      throw new DomainError("No se pueden sumar montos de distintas monedas.");
    return new Money(this.value + other.value, this.currency);
  }

  equals(other: Money): boolean {
    return this.value === other.value && this.currency === other.currency;
  }
}

// Entidad Order ahora usa el VO
class Order {
  private amount: Money; // ← el VO protege sus propias invariantes

  constructor(amount: Money) {
    this.amount = amount;
  }
}
```

**Candidates frecuentes para Value Objects:**

| Primitivo                | Value Object sugerido       | Invariantes típicas                    |
|--------------------------|-----------------------------|----------------------------------------|
| `number` (dinero)        | `Money`                     | No negativo, moneda válida (ISO)       |
| `string` (email)         | `Email`                     | Formato `x@x.x`, longitud máxima       |
| `string` (teléfono)      | `PhoneNumber`               | Formato internacional, dígitos válidos |
| `number` (porcentaje)    | `Percentage`                | Entre 0 y 100                          |
| `string[]` (dirección)   | `Address`                   | Campos requeridos, longitudes          |
| `string` (UUID)          | `UserId`, `OrderId`, etc.   | Formato UUID válido                    |

---

### 2.2 Modelo de Dominio Anémico

**Señal de alerta:** Las entidades solo tienen atributos con `getters` y `setters`, sin ningún método con lógica de negocio.

#### ❌ Violación

```typescript
class User {
  private name: string;
  private age: number;
  private status: string;

  getName() { return this.name; }
  setName(name: string) { this.name = name; }
  getAge() { return this.age; }
  setAge(age: number) { this.age = age; }   // ← acepta -5, 300, lo que sea
  getStatus() { return this.status; }
  setStatus(status: string) { this.status = status; } // ← acepta "ZOMBIE"
}

// Application Service (cargado de reglas de negocio que no le pertenecen):
class UserService {
  updateAge(userId: string, newAge: number) {
    if (newAge < 0 || newAge > 120) throw new Error("Edad inválida");
    const user = this.userRepo.findById(userId);
    user.setAge(newAge);
    this.userRepo.save(user);
  }
}
```

#### ✅ Refactorización: Modelo Rico

```typescript
class User {
  private name: string;
  private age: number;
  private status: UserStatus; // ← enum de dominio, no string libre

  // El comportamiento vive en la entidad
  updateAge(newAge: number): void {
    if (newAge < 0 || newAge > 120)
      throw new DomainError(`Edad fuera de rango permitido: ${newAge}`);
    this.age = newAge;
  }

  activate(): void {
    if (this.status === UserStatus.ACTIVE)
      throw new DomainError("El usuario ya está activo.");
    this.status = UserStatus.ACTIVE;
  }

  deactivate(): void {
    this.status = UserStatus.INACTIVE;
  }
}

// Application Service simplificado y limpio:
class UserApplicationService {
  updateAge(userId: string, newAge: number) {
    const user = this.userRepo.findById(userId); // recuperar
    user.updateAge(newAge);                       // delegar al dominio
    this.userRepo.save(user);                     // persistir
  }
}
```

---

### 2.3 Invariantes no Protegidas

**Señal de alerta:** El constructor de una entidad o Value Object acepta cualquier valor sin validación.

#### ❌ Violación

```typescript
class Product {
  constructor(
    public id: string,
    public name: string,  // puede ser ""
    public price: number, // puede ser -100
    public stock: number  // puede ser -999
  ) {}
}
```

#### ✅ Refactorización

```typescript
class Product {
  private readonly id: ProductId;
  private name: string;
  private price: Money;
  private stock: number;

  constructor(id: ProductId, name: string, price: Money, stock: number) {
    if (!name || name.trim().length === 0)
      throw new DomainError("El nombre del producto es obligatorio.");
    if (stock < 0)
      throw new DomainError("El stock inicial no puede ser negativo.");

    this.id    = id;
    this.name  = name.trim();
    this.price = price;   // Money ya valida sus propias reglas
    this.stock = stock;
  }

  reduceStock(quantity: number): void {
    if (quantity <= 0) throw new DomainError("La cantidad debe ser positiva.");
    if (this.stock < quantity) throw new DomainError("Stock insuficiente.");
    this.stock -= quantity;
  }
}
```

---

## 3. Violaciones de Agregados

### 3.1 Acceso Directo a Entidades Internas

**Señal de alerta:** Código externo al agregado modifica directamente sus entidades internas sin pasar por la Raíz.

#### ❌ Violación

```typescript
class PurchaseOrder {         // Aggregate Root
  public orderLines: OrderLine[]; // ← público, sin control
}

// En Application Service (violación de la frontera del agregado):
const line = order.orderLines[0]; // acceso directo
line.setQuantity(5);              // mutación sin pasar por la raíz
line.setPrice(-100);              // ninguna invariante del agregado se verifica
repo.save(order);
```

#### ✅ Refactorización

```typescript
class PurchaseOrder {          // Aggregate Root
  private orderLines: OrderLine[]; // ← privado

  addLine(productId: ProductId, quantity: number, unitPrice: Money): void {
    if (quantity <= 0) throw new DomainError("Cantidad inválida.");
    const line = new OrderLine(productId, quantity, unitPrice);
    this.orderLines.push(line);
  }

  updateLineQuantity(productId: ProductId, newQuantity: number): void {
    const line = this.orderLines.find(l => l.productId.equals(productId));
    if (!line) throw new DomainError("Línea de orden no encontrada.");
    line.updateQuantity(newQuantity); // la raíz delega controladamente
  }

  // Lectura: exponer copia inmutable o DTO, nunca la referencia directa
  getLines(): ReadonlyArray<OrderLineSnapshot> {
    return this.orderLines.map(l => l.toSnapshot());
  }
}
```

---

### 3.2 Aggregate Root sin Responsabilidades Reales

**Señal de alerta:** La raíz del agregado no valida ninguna invariante; solo delega mecánicamente sin lógica.

**Criterio de revisión:** Pregunta "¿Qué regla de negocio se rompería si esto fuera un setter directo?" — si la respuesta es "ninguna", revisa si realmente es un agregado o si la lógica fue evacuada a un servicio.

```typescript
// ❌ Raíz vacía de significado
class ShoppingCart {
  addItem(item: CartItem) {
    this.items.push(item); // no verifica límites, duplicados, stock, etc.
  }
}

// ✅ Raíz que cumple su rol de guardián de invariantes
class ShoppingCart {
  private static MAX_ITEMS = 20;

  addItem(productId: ProductId, quantity: number, price: Money): void {
    if (this.items.length >= ShoppingCart.MAX_ITEMS)
      throw new DomainError("El carrito ha alcanzado el límite de productos.");

    const existingItem = this.items.find(i => i.productId.equals(productId));
    if (existingItem) {
      existingItem.increaseQuantity(quantity); // actualiza si ya existe
    } else {
      this.items.push(new CartItem(productId, quantity, price));
    }
  }
}
```

---

## 4. Violaciones de Capa de Aplicación

### 4.1 Lógica de Negocio en Application Services

**Señal de alerta:** El Application Service contiene `if`/`else` con reglas que el experto del negocio reconocería como propias del dominio.

#### ❌ Violación

```typescript
class OrderApplicationService {
  placeOrder(customerId: string, items: ItemDto[]) {
    const customer = this.customerRepo.findById(customerId);

    // ← regla de negocio viviendo en la capa de aplicación
    if (customer.getDebt() > 5000) {
      throw new Error("Cliente con deuda excesiva, no puede comprar.");
    }

    // ← cálculo de negocio en lugar incorrecto
    const total = items.reduce((sum, i) => sum + i.price * i.qty, 0);
    if (total < 10) throw new Error("Orden mínima no alcanzada.");

    // ...
  }
}
```

#### ✅ Refactorización

```typescript
// La regla vive en la entidad Customer
class Customer {
  canPlaceOrder(): boolean {
    return this.debt.isLessThanOrEqual(new Money(5000, "USD"));
  }
}

// El cálculo y la validación del mínimo vive en el agregado Order
class Order {
  private static MIN_ORDER = new Money(10, "USD");

  static create(customer: Customer, lines: OrderLine[]): Order {
    if (!customer.canPlaceOrder())
      throw new DomainError("Cliente no habilitado para realizar pedidos.");

    const total = lines.reduce((sum, l) => sum.add(l.subtotal()), Money.zero("USD"));
    if (total.isLessThan(Order.MIN_ORDER))
      throw new DomainError("El pedido no alcanza el monto mínimo.");

    return new Order(customer.id, lines, total);
  }
}

// Application Service limpio: solo orquesta
class OrderApplicationService {
  placeOrder(customerId: string, items: ItemDto[]) {
    const customer = this.customerRepo.findById(customerId);     // 1. Recuperar
    const lines    = this.buildLines(items);                     // 2. Preparar datos
    const order    = Order.create(customer, lines);              // 3. Delegar al dominio
    this.orderRepo.save(order);                                  // 4. Persistir
    this.eventBus.publish(order.pullDomainEvents());             // 5. Publicar eventos
  }
}
```

---

### 4.2 Fuga de Infraestructura hacia Application / Domain

**Señal de alerta:** Aparecen imports de frameworks de persistencia, ORM, clientes HTTP, o lógica SQL directamente en clases de dominio o servicios de aplicación.

#### ❌ Violación

```typescript
import { getConnection } from "typeorm";   // ← infraestructura en la capa de dominio

class OrderDomainService {
  calculateDiscount(orderId: string): Money {
    const conn = getConnection();
    const raw  = conn.query(`SELECT * FROM orders WHERE id = '${orderId}'`); // ← SQL en dominio
    // ...
  }
}
```

#### ✅ Refactorización: Inversión de Dependencia (DIP)

```typescript
// 1. Definir el puerto (interfaz) en la capa de dominio
interface OrderRepository {
  findById(id: OrderId): Order | null;
  save(order: Order): void;
}

// 2. Implementar el adaptador en infraestructura
class TypeOrmOrderRepository implements OrderRepository {
  findById(id: OrderId): Order | null {
    // Aquí sí puede haber TypeORM, SQL, etc.
    return this.ormRepo.findOne({ where: { id: id.value } });
  }
  save(order: Order): void { /* ... */ }
}

// 3. El dominio solo conoce la interfaz, nunca la implementación
class OrderApplicationService {
  constructor(private orderRepo: OrderRepository) {} // inyección de dependencia
}
```

---

## 5. Violaciones en el Manejo de Servicios Externos

Esta categoría es especialmente crítica porque las fallas en servicios externos (APIs de pago, notificaciones, servicios de terceros) tienen comportamientos impredecibles: timeouts, errores de red, respuestas inesperadas.

### 5.1 Excepciones Técnicas Expuestas al Dominio

**Señal de alerta:** Los errores de red, HTTP o de base de datos se propagan sin transformar hasta la capa de dominio o se exponen directamente a los consumidores de la API.

#### ❌ Violación

```typescript
class PaymentApplicationService {
  processPayment(orderId: string, cardToken: string) {
    // Si Stripe falla, la excepción nativa de la librería se propaga sin control
    const result = this.stripeClient.chargeCard(cardToken, amount);
    // StripeInvalidRequestError, NetworkError, etc. → llegan sin transformar
  }
}
```

**Problemas:**
- El dominio queda acoplado a los tipos de error de Stripe.
- Si se cambia de proveedor de pago, hay que buscar y cambiar el manejo de errores en toda la aplicación.
- El cliente recibe mensajes técnicos internos (stack traces, códigos de la librería).

#### ✅ Refactorización: Puerto + Adaptador + Errores de Dominio

```typescript
// 1. Puerto en capa de dominio / aplicación — sin rastro de Stripe
interface PaymentGateway {
  charge(amount: Money, cardToken: string): PaymentResult;
}

// 2. Errores de dominio propios (sin acoplamiento a libs externas)
class PaymentDeclinedError extends DomainError {
  constructor(reason: string) {
    super(`Pago rechazado: ${reason}`);
  }
}
class PaymentGatewayUnavailableError extends DomainError {
  constructor() { super("El servicio de pagos no está disponible. Intente más tarde."); }
}

// 3. Adaptador en infraestructura: absorbe y traduce los errores técnicos
class StripePaymentAdapter implements PaymentGateway {
  charge(amount: Money, cardToken: string): PaymentResult {
    try {
      const response = this.stripeClient.chargeCard(cardToken, amount.value);
      return PaymentResult.success(response.transactionId);

    } catch (error) {
      if (error instanceof Stripe.errors.StripeCardError) {
        // Error conocido de negocio → error de dominio propio
        throw new PaymentDeclinedError(error.message);
      }
      if (error instanceof Stripe.errors.StripeConnectionError) {
        // Error técnico → error de dominio genérico sin detalles internos
        throw new PaymentGatewayUnavailableError();
      }
      // Error inesperado → loguear internamente, exponer mensaje seguro
      this.logger.error("Error inesperado en Stripe", error);
      throw new PaymentGatewayUnavailableError();
    }
  }
}

// 4. Application Service sin ningún conocimiento de Stripe
class PaymentApplicationService {
  constructor(private gateway: PaymentGateway) {} // inyecta el adaptador

  processPayment(orderId: string, cardToken: string): void {
    const order  = this.orderRepo.findById(new OrderId(orderId));
    const result = this.gateway.charge(order.total, cardToken); // errores ya son del dominio
    order.markAsPaid(result.transactionId);
    this.orderRepo.save(order);
  }
}
```

---

### 5.2 Ausencia de Capa Anticorrupción (ACL)

**Señal de alerta:** Los modelos de datos de APIs externas o sistemas legados se usan directamente dentro del dominio propio.

#### ❌ Violación

```typescript
// El dominio importa y usa directamente el modelo de la API externa
import { ShipmentTrackingResponse } from "external-logistics-sdk";

class Order {
  updateTracking(tracking: ShipmentTrackingResponse) { // ← modelo externo en el dominio
    this.status = tracking.shipment_status_code;  // ← vocabulario del externo, no del nuestro
  }
}
```

**Problema:** Si el proveedor logístico cambia su API o se reemplaza, el modelo del dominio se contamina y debe cambiar también.

#### ✅ Refactorización: Anti-Corruption Layer

```typescript
// 1. Modelo propio del dominio (Lenguaje Ubicuo propio)
enum ShipmentStatus { IN_TRANSIT, DELIVERED, RETURNED }

class ShipmentInfo {
  constructor(
    readonly trackingCode: string,
    readonly status: ShipmentStatus,
    readonly estimatedDelivery: Date
  ) {}
}

// 2. ACL en infraestructura: traduce el vocabulario externo al nuestro
class LogisticsAntiCorruptionLayer {
  translate(raw: ShipmentTrackingResponse): ShipmentInfo {
    const statusMap: Record<string, ShipmentStatus> = {
      "IN_TRANSIT": ShipmentStatus.IN_TRANSIT,
      "DLVRD":      ShipmentStatus.DELIVERED,
      "RTN":        ShipmentStatus.RETURNED,
    };

    const status = statusMap[raw.shipment_status_code];
    if (!status) throw new Error(`Estado de envío desconocido: ${raw.shipment_status_code}`);

    return new ShipmentInfo(
      raw.tracking_id,
      status,
      new Date(raw.est_delivery_timestamp)
    );
  }
}

// 3. El dominio trabaja solo con su propio modelo
class Order {
  updateShipment(info: ShipmentInfo): void { // ← modelo propio, no externo
    this.shipmentStatus = info.status;
    this.trackingCode   = info.trackingCode;
  }
}
```

---

## 6. Violaciones de la Regla de Dependencia

**Señal de alerta:** Una clase del dominio tiene un `import` que apunta a infraestructura, o un repositorio depende directamente de entidades de dominio en lugar de interfaces.

```
Escenarios de violación más comunes:

  Domain    →   Infrastructure      ❌ (el dominio conoce TypeORM, Stripe, etc.)
  Domain    →   Application         ❌ (el dominio conoce Application Services)
  Application → Infrastructure      ⚠️  (permitido solo a través de interfaces/puertos)
```

| Situación detectada                                      | Capa afectada | Corrección                                      |
|----------------------------------------------------------|---------------|-------------------------------------------------|
| `import { Repository } from 'typeorm'` en una Entidad   | Dominio       | Definir interfaz en dominio, implementar en infra |
| `import axios from 'axios'` en un Domain Service        | Dominio       | Crear puerto `HttpClient`, adaptador en infra   |
| `import { OrderEntity } from '../domain'` en repositorio | Infra       | Repositorio recibe la interfaz, no la clase concreta |
| Controlador REST llama directamente a un Repositorio    | Infra         | El controlador solo llama al Application Service |

---

## 7. Tabla de Detección Rápida

| Síntoma observado en el código                                    | Violación probable                      | Sección |
|-------------------------------------------------------------------|-----------------------------------------|---------|
| Atributos `int`, `string`, `float` con reglas asociadas           | Obsesión Primitiva                      | 2.1     |
| Entidad solo con `get` / `set`, validaciones en el servicio       | Modelo Anémico                          | 2.2     |
| Constructor acepta cualquier valor sin lanzar excepciones         | Invariantes no protegidas               | 2.3     |
| `orderLines[i].setX(...)` desde fuera del Aggregate Root          | Bypass del Aggregate Root               | 3.1     |
| `if (x > y) throw` en un Application Service                     | Lógica de negocio en capa equivocada    | 4.1     |
| `import typeorm / axios / sql` en dominio o aplicación            | Fuga de infraestructura                 | 4.2     |
| `catch (StripeError)` en Application Service o Domain             | Errores técnicos sin transformar        | 5.1     |
| Modelo de API externa usado como parámetro en entidades propias   | Sin capa anticorrupción (ACL)           | 5.2     |

---

## 8. Checklist de Refactorización

Antes de modificar una clase incorrecta, responde estas preguntas en orden:

**Paso 1 — Identificar a qué capa pertenece la clase:**
- [ ] ¿Es lógica pura del negocio? → Dominio
- [ ] ¿Orquesta el dominio con la infraestructura? → Aplicación
- [ ] ¿Es un detalle técnico (BD, API, UI)? → Infraestructura

**Paso 2 — Revisar si los tipos de datos son correctos:**
- [ ] ¿Hay primitivos que representan conceptos del negocio? → Extraer Value Object
- [ ] ¿El Value Object valida sus invariantes en el constructor?

**Paso 3 — Verificar que la entidad / agregado tenga comportamiento:**
- [ ] ¿La entidad tiene métodos con lógica de negocio (no solo setters)?
- [ ] ¿El Aggregate Root controla el acceso a sus entidades internas?
- [ ] ¿Los métodos del agregado validan invariantes antes de mutar el estado?

**Paso 4 — Revisar la capa de aplicación:**
- [ ] ¿El Application Service solo orquesta (recuperar → delegar → persistir → publicar)?
- [ ] ¿No hay `if` con lógica de negocio que debería estar en el dominio?

**Paso 5 — Revisar manejo de servicios externos:**
- [ ] ¿El adaptador de infraestructura captura todos los errores técnicos?
- [ ] ¿Los errores técnicos se traducen a errores de dominio propios?
- [ ] ¿Los modelos externos pasan por una ACL antes de entrar al dominio?

**Paso 6 — Verificar la Regla de Dependencia:**
- [ ] ¿El dominio depende solo de interfaces (no de implementaciones concretas)?
- [ ] ¿La infraestructura implementa las interfaces definidas por el dominio?
- [ ] ¿El dominio tiene cero imports de librerías de infraestructura?

---

*Guía elaborada sobre la base de los apuntes del Parcial #2 — DDD + Arquitectura CLEAN/Hexagonal.*
*Referencia de clase: 11-05-2026 al 05-06-2026.*

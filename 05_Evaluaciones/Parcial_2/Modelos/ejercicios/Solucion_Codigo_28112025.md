
# 🔍 Ejercicio de Refactorización: Architectural Forensics (Caso TypeScript)

> [!danger] Code Smell Detectado: Fat Controller & Control Freak
> El siguiente componente viola el principio de inversión de dependencias y presenta obsesión primitiva, acoplando la presentación directamente con el proveedor de pagos y el sistema de archivos.

---

## 💻 Código Fuente Original (Legado)

```typescript
// BillingController.ts
export class BillingController {
    pay(amount: number): number {
        if (amount <= 0) {
            return 0;
        }
        
        const gateway = new StripeGateway("secret-key");
        const logger = new FileLogger("log.txt");
        
        const result = gateway.process(amount);
        logger.log("done");
        
        return result ? amount : 0;
    }
}
````

## 🔍 Diagnóstico de Ingeniería (Code Smells & Violaciones)

### 1. Antipadrón Control Freak y Fuga de Infraestructura

El controlador utiliza explícitamente la palabra clave `new` para instanciar `StripeGateway` y `FileLogger`.

- **Problema:** Al crear las dependencias internamente, el controlador queda "encadenado" estructuralmente a Stripe y al sistema de archivos local (`log.txt`). Esto destruye la **testeabilidad** (no se pueden meter mocks) e impide la mantenibilidad si se decide migrar a PayPal o a un logger en la nube (ej. CloudWatch).
- **Solución:** Aplicar Inversión de Dependencias (DIP) mediante Inyección por Constructor, requiriendo abstracciones/interfaces (`IPaymentGateway` e `ILogger`).

### 2. Violación del Principio de Responsabilidad Única (SRP)

La clase actúa como un "God Object" a nivel de método: valida el input, quema credenciales confidenciales, orquesta el flujo de persistencia/red y toma decisiones de negocio.

- **Problema:** Múltiples vectores de cambio (cambio de pasarela, cambio de formato de logs, cambio en las políticas de cobro).
- **Solución:** Extraer la lógica de negocio a un **Caso de Uso / Servicio de Aplicación** (`ProcessPaymentUseCase`), reduciendo el controlador a un mero punto de entrada de la presentación.

### 3. Obsesión Primitiva e Invariantes Desprotegidas

El parámetro `amount` se maneja como un tipo básico de TypeScript (`number`).

- **Problema:** La regla de negocio `if (amount <= 0)` queda expuesta en la capa de presentación. Esto genera un **Modelo de Dominio Anémico**, permitiendo que otras capas de la aplicación manipulen o propaguen montos inválidos accidentalmente.
- **Solución:** Encapsular el concepto en un _Value Object_ `Money` que proteja su invariante estructural de forma atómica en su propio constructor.

### 4. Configuración Embebida (Hardcoded Strings)

La clave `"secret-key"` y el archivo `"log.txt"` están grabados en duro en el flujo de ejecución.

- **Problema:** Riesgo crítico de seguridad (fuga de credenciales en el repositorio de Git) e incapacidad de desacoplar el software por entornos (Desarrollo, QA, Producción).
- **Solución:** Externalizar la configuración hacia variables de entorno (`process.env`) e inyectarlas desde la Raíz de Composición (_Composition Root_).

## 🛠️ Plan de Refactorización (Diseño del Target Architecture)

### Mapeo de Responsabilidades por Capa

|**Componente Actual**|**Violación Detectada**|**Reemplazo DDD / Clean Arq.**|**Capa de Destino**|
|---|---|---|---|
|`number amount`|Obsesión Primitiva|`Money` (Value Object)|**Dominio**|
|`if (amount <= 0)`|Lógica en capa incorrecta|Validación interna de `Money`|**Dominio**|
|`new StripeGateway()`|Control Freak / Fuga|`IPaymentGateway` (Puerto)|**Dominio / Aplicación**|
|Orquestación de `pay`|Ausencia de Caso de Uso|`ProcessPaymentUseCase`|**Aplicación**|
|`BillingController`|Falta de SRP / Fat Controller|Controlador Delgado (Delegación)|**Presentación**|

## 📐 Lienzo de Diseño Táctico (Para Resolver)

_Utiliza este espacio en blanco para estructurar los límites y contratos de los componentes limpios extraídos:_

### Value Objects a Diseñar

|**Nombre del Value Object**|**Atributos**|**Invariantes a Validar (Constructor)**|
|---|---|---|
||||

### Puertos / Interfaces de Aplicación

|**Nombre de la Interfaz (Puerto)**|**Métodos Definidos**|**Capa de la Interfaz**|**Ubicación de la Implementación**|
|---|---|---|---|
|||||
|||||

### Casos de Uso (Servicios de Aplicación)

|**Nombre del Caso de Uso**|**Dependencias Requeridas (Inyectadas)**|**Operación Principal**|
|---|---|---|
||||

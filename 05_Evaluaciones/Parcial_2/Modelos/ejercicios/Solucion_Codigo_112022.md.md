
# 🔍 Ejercicio de Refactorización: Architectural Forensics (Caso .NET / DDD Táctico)

> [!danger] Code Smell Detectado: Anemic Domain & Active Record Leak
> El siguiente componente presenta lógica de negocio dispersa en servicios (orquestadores gordos), violación de persistencia al delegar el guardado a las entidades, y contaminación por preocupaciones transversales (Logging/Alertas).

---

## 💻 Código Fuente Original (Legado)

```csharp
// TicketAssignmentService.cs (Fragmento de ejecución interna)
_alerts.Send(agent, "You have a new ticket!");
_logger.debug("Se creo un ticket con id=" + ticket.Id + " y asignado a " + agent.toString());
}
catch(Exception e)
{
    _logger.error(e);
}
// [Contexto implícito del fragmento]:
// agent.ActiveTickets = agent.ActiveTickets + 1;
// agent.Save();
// ticket.Save();
````

## 🔍 Diagnóstico de Ingeniería (Code Smells & Violaciones)

### 1. Modelo de Dominio Anémico

Las entidades `Agent` y `Ticket` son tratadas como simples contenedores de datos (_Property Bags_). La operación mutadora `agent.ActiveTickets = agent.ActiveTickets + 1` altera el estado desde afuera.

- **Problema:** La lógica de negocio está expuesta en el servicio de aplicación, permitiendo estados inconsistentes.
- **Solución:** Encapsular el comportamiento dentro de la entidad rica `Agent` mediante un método explícito (ej. `agent.AssignTicket(ticket)`), mutando el estado de forma interna y controlada.

### 2. Violación de Persistencia (Antipatrón Active Record)

El uso de llamadas directas como `agent.Save()` y `ticket.Save()` indica que los objetos de negocio conocen el mecanismo de base de datos o el ORM subyacente.

- **Problema:** Acoplamiento severo del dominio con la capa de infraestructura, lo que imposibilita realizar pruebas unitarias puras en memoria.
- **Solución:** Implementar el patrón **Repository**. La persistencia es una abstracción externa; las entidades se pasan a interfaces como `IAgentRepository.Save(agent)`.

### 3. Antipatrón Control Freak (Instanciación Directa)

Hacer `new Ticket()` dentro de la lógica del servicio viola el principio de Inversión de Control (IoC).

- **Problema:** Impide el uso de _Factories_ que aseguren que el objeto de dominio nazca con sus invariantes estructurales válidas y complica el aislamiento en los tests.
- **Solución:** Delegar la instanciación a un método de fábrica o permitir que el propio Agregado genere el objeto derivado.

### 4. Fuga de Infraestructura y Preocupaciones Transversales (Cross-Cutting Concerns)

El bloque `try/catch`, el manejo de excepciones y la infraestructura de logging (`_logger.debug`) o alertas (`_alerts.Send`) contaminan la pureza de la lógica de negocio.

- **Problema:** Violación del principio de Separación de Incumbencias (SoC).
- **Solución:** Extraer estas preocupaciones transversales hacia la frontera de la capa de aplicación o encapsularlas utilizando **Programación Orientada a Aspectos (AOP)** o el patrón **Decorator**.

### 5. Invariantes No Protegidas

- **Problema:** No existen reglas de validación en los límites críticos (ej. validar si el agente superó el umbral máximo de tickets permitidos en paralelo). Al estar la lógica suelta en el servicio, cualquier otro endpoint podría saltarse las reglas del negocio.
- **Solución:** Centralizar la regla dentro de la Raíz del Agregado para forzar la consistencia antes de mutar los campos.

## 🛠️ Plan de Refactorización (Diseño del Target Architecture)

### Mapeo de Responsabilidades por Capa

|**Lógica / Línea Original**|**Patrón DDD / Clean Arq.**|**Capa de Destino**|
|---|---|---|
|`agent.Save()` / `ticket.Save()`|`IAgentRepository` / `ITicketRepository` (Puertos)|**Infraestructura (Impl) / Aplicación (Interface)**|
|`ActiveTickets = ActiveTickets + 1`|Comportamiento en Entidad Rica `Agent`|**Dominio**|
|`_logger.debug()` / `try-catch`|Interceptores / Decoradores AOP|**Infraestructura / Aplicación (Frontera)**|
|`_alerts.Send()`|`IEventNotificationService` (Puerto de salida)|**Aplicación (Interface)**|
|Flujo: Recuperar ➔ Asignar ➔ Guardar|Caso de Uso (`AssignTicketUseCase`)|**Aplicación**|

## 📐 Lienzo de Diseño Táctico (Para Resolver)

_Utiliza este espacio en blanco para estructurar el modelo de dominio rico y los contratos de persistencia desacoplados:_

### Métodos de Entidad Rica (Comportamiento y Reglas)

|**Entidad / Agregado Root**|**Método de Negocio**|**Invariantes / Validaciones a Proteger**|
|---|---|---|
||||

### Repositorios e Interfaces (Puertos)

|**Nombre del Puerto / Interfaz**|**Firma del Método**|**Propósito de Aislamiento**|
|---|---|---|
||||
||||

### Flujo del Caso de Uso (Orquestador Limpio)

Plaintext

```
[Caso de Uso: AssignTicketUseCase]
  ├── 1. Recibir Input DTO (agentId, ticketId)
  ├── 2. Llamar a Repository para recuperar el Agregado Root: ______________________
  ├── 3. Ejecutar el método de negocio sobre el Agregado: __________________________
  ├── 4. Pasar el Agregado modificado al Repository para persistir: ________________
  └── 5. Despachar eventos de dominio (opcional)
```

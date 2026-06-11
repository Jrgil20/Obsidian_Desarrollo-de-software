# 📦 Workspace: Caso Práctico PharmaLogistics S.A.

> [!abstract] Índices del Repositorio
> Espacio de trabajo modular para el diseño del modelo táctico de DDD de PharmaLogistics S.A. enfocado en la preservación estricta de la cadena de frío, consistencia transaccional y aislamiento de infraestructura.

---

## 🏛️ Diseño del Modelo Táctico (Domain-Driven Design)

### 1. Value Objects Enriquecidos (Rich Value Objects)

```text
[Value Object: TemperatureRange]
  ├── Atributos: Min (Temperature), Max (Temperature)
  ├── Método: Intersect(TemperatureRange other) -> Devuelve el rango común más estricto
  └── Método: Covers(TemperatureRange needed) -> Evalúa si el rango operativo contiene al requerido
````

- **`Temperature` (Micro-type):** Encapsula un valor numérico (`double`).
    
    - _Invariantes:_ El constructor valida y lanza una excepción si el valor es inferior al cero absoluto ($-273.15^\circ\text{C}$).
    - _Responsabilidades:_ Provee métodos semánticos como `IsHigherThan(Temperature)` y `IsLowerThan(Temperature)`.
        
- **`TemperatureRange` (Composición):** Compuesto por dos objetos `Temperature` (`Min` y `Max`).
    
    - _Invariantes:_ El constructor valida de forma atómica que `Min` no sea mayor que `Max`.
        
    - _Responsabilidades:_
	       1. `Intersect(TemperatureRange other)`: Calcula el rango más estricto entre medicamentos; lanza `IncompatibleMedicinesException` si no hay solapamiento.
        2. `Covers(TemperatureRange other)`: Valida si un rango operativo abarca por completo a otro.
        3. `Contains(Temperature)`: Verifica si una lectura puntual cae dentro de los límites seguros.
        
- **`Volume` (Unidad de Medida):** Encapsula el valor numérico y la escala estándar (ej. Litros).
    
    - _Invariantes:_ Debe ser estrictamente mayor a cero.
    - _Responsabilidades:_ `IsLargerThan(Volume other)` para comparaciones de capacidad sin exponer primitivos.

- **`SensorReading` (Hecho Histórico Inmutable):** Compuesto por un `Instant` (Timestamp) y una `Temperature`.
    
    - _Invariantes:_ Al representar un evento físico consumado, carece de mutadores (_setters_). Es completamente inmutable.

### 2. Límites de Agregados y Entidades

#### 📦 Agregado: Shipment (Envío)

- **Raíz del Agregado (Aggregate Root):** `Shipment`
    
    - _Atributos:_ `ShipmentId`, `ShipmentStatus` (PENDIENTE, EN_TRANSITO, COMPROMETIDO), `AssignedContainerId` (ID referencial, no el objeto).
    - _Entidades Internas:_ `Medicine` (Colección dentro del agregado). Posee `MedicineId`, `Name`, `Volume` y `TemperatureRange`.
        
    - _Invariantes Globales de Consistencia:_
        
        1. El `RequiredRange` del envío se deriva dinámicamente orquestando la intersección de todos los rangos de su lista de medicamentos.
        2. Un envío no puede pasar al estado `EN_TRANSITO` si no tiene un `AssignedContainerId` previamente validado.


#### 📦 Agregado: Container (Contenedor)

- **Raíz del Agregado (Aggregate Root):** `Container`
    
    - _Atributos:_ `ContainerId`, `MaxVolume` (Volume), `OperatingTemperatureRange` (TemperatureRange), `ContainerStatus` (DISPONIBLE, EN_TRANSITO, EN_MANTENIMIENTO, COMPROMETIDO).
    - _Invariantes Globales de Consistencia:_
        
        1. **Regla Inviolable:** Al procesar una `SensorReading`, si `OperatingTemperatureRange.Contains(reading.Temperature)` es falso, el estado muta inmediatamente a `COMPROMETIDO` y se dispara el evento de dominio `ContainerCompromisedEvent`.
        2. Un contenedor solo acepta cargas si `MaxVolume.IsLargerThan(shipmentVolume)` y su `OperatingTemperatureRange.Covers(shipmentRange)`.
    

### 3. Servicios de Dominio (Pureza de Procesos Stateless)

- **`AssignContainerToShipmentService` (Proceso 1):** Orquesta la asignación de transporte.
    
    - _Flujo:_ Recupera el rango e ingresos de carga del `Shipment`, consulta el repositorio de contenedores en estado `DISPONIBLE` y delega en el método del contenedor la validación de si puede soportar el volumen y el rango térmico requerido.

- **`ContainerMonitoringService` (Procesos 2 y 3):** Maneja la ingesta de telemetría y consistencia eventual.
    
    - _Flujo:_ Registra la nueva lectura en el agregado `Container`. Si el contenedor dispara un `ContainerCompromisedEvent`, el servicio intercepta el evento, localiza mediante consultas al repositorio el `Shipment` activo vinculado a ese ID de contenedor, y fuerza la mutación del estado del envío a `COMPROMETIDO`.

- **`ShipmentDepartureValidator` (Proceso 4):** Validación final previa al despacho en aduana.
    
    - _Flujo:_ Verifica que el contenedor asignado mantenga el estado `DISPONIBLE` en el sistema y consume el puerto de salida `IRegulacionesRepository` para validar la legalidad jurídica de la ruta basándose en la lista de medicamentos del envío.


## 🛠️ Plan de Refactorización (Target Architecture Matrix)

|**Requisito / Proceso**|**Patrón DDD / Clean Arq.**|**Capa de Destino**|
|---|---|---|
|Evaluación de Rango Térmico (`Intersect`)|Comportamiento en VO (`TemperatureRange`)|**Dominio**|
|Registro de Telemetría y Cambio de Estado|Lógica interna del AR (`Container`)|**Dominio**|
|Reacción en Cadena: Contenedor ➔ Envío|Servicio de Dominio + Eventos de Dominio|**Dominio**|
|Consulta Externa `IRegulacionesRepository`|Puerto (Interfaz de Infraestructura)|**Aplicación (Interface)**|
|Endpoint / Orquestador Inicial|Controlador / Caso de Uso correspondiente|**Presentación / Aplicación**|

## 📐 Lienzo de Diseño Táctico (Para Resolver en Simulacro)

### Ejercicio 1: Modelado de Métodos de Negocio en Código

_Utiliza este bloque para escribir el pseudo-código o código de producción del método `Intersect` del Value Object o la máquina de estados del Contenedor:_

C#

```
// Escribe aquí tu propuesta de implementación limpia (ej. C# / TypeScript)

```

### Ejercicio 2: Trazado de Interfaces de Persistencia e Infraestructura (Puertos)

|**Nombre del Puerto / Interfaz**|**Firma del Método Requerido**|**Propósito Arquitectónico**|
|---|---|---|
||||
||||

### Ejercicio 3: Flujo de Ejecución del Proceso 3 (Diagrama de Texto / Secuencia)

Plaintext

```
[Sensor Ingest Endpoint] ──> Dispacha Lectura
    ├── 1. Container AR: Evalúa lectura y muta a COMPROMETIDO
    ├── 2. Domain Event: _________________________________________________
    ├── 3. Domain Service: _______________________________________________
    └── 4. Shipment AR: Muta estado de la carga a COMPROMETIDO
```


***

### 💡 Tip de examen para este caso:
Presta especial atención al **Proceso 3**. En los exámenes de la UCAB, la reacción en cadena donde un agregado modifica a otro suele evaluarse de dos formas: mediante un **Servicio de Dominio** que tiene acceso a ambos repositorios en la misma transacción, o mediante **Eventos de Dominio** para mantener la consistencia eventual si fuesen microservicios separados. El lienzo de diseño de arriba está preparado para que practiques ambas estrategias.

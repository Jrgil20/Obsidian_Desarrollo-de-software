# 📦 Workspace: Caso Práctico Gestión de Eventos Corporativos y Sociales

> [!abstract] Índices del Repositorio
> Espacio de trabajo modular para el diseño del modelo táctico de DDD del sistema de organización de eventos. El diseño se rige estrictamente bajo la restricción mandatoria de cuatro (4) agregados y la separación de reglas de negocio empresariales y de aplicación.

---

## 🏛️ Restricciones del Modelo de Dominio (Mandatorio)

El sistema se acopla exclusivamente a los siguientes límites transaccionales:
* 📦 **Agregado Evento:** Raíz `Evento`. Controla el ciclo de vida, la ubicación, fecha, aforo y la lista de inscripciones/espera.
* 📦 **Agregado Participante:** Raíz `Participante`. Protege los datos de identidad (Cédula, nombres, contacto).
* 📦 **Agregado Catering:** Raíz `Catering`. Gestiona la cotización externa, cantidades y tipos de comida.
* 📦 **Agregado Audiovisual:** Raíz `Audiovisual`. Controla el alquiler de equipos, soporte técnico y logística AV.

---

## ⚙️ Identificación y Justificación de Servicios de Dominio (Stateless)

### 1. `CalculadorCostosEvento` (EventCostCalculator)
* **Responsabilidad:** Calcular el monto total facturable del evento. Orquesta la recolección de los costos base del `Evento`, los conceptos del agregado `Catering` (comidas/pasapalos) y los contratos del agregado `Audiovisual` (equipos/personal), aplicando multiplicadores según la cantidad necesaria.
* **Justificación de Dominio:** En DDD, un agregado no debe conocer las interioridades de otro ni cargar con entidades de un dominio ajeno (solo se referencian por ID). Como la regla de cálculo requiere cruzar datos financieros de tres agregados independientes (`Evento`, `Catering`, `Audiovisual`), la lógica no pertenece de forma natural a ninguno de ellos por sí solo. El servicio ejecuta esta operación pura y *stateless* sin violar las fronteras de consistencia.

### 2. `GestorInscripcionesServicio` (InscriptionManagerService)
* **Responsabilidad:** Validar los flujos de negocio del proceso de registro. Controla que el `Participante` cumpla las condiciones, evalúa el aforo máximo del `Evento`, gestiona el encolamiento en la lista de espera y automatiza la promoción de un participante en espera al estado "Confirmado" de forma atómica cuando ocurre una cancelación.
* **Justificación de Dominio:** Modificar el estado de una inscripción requiere alterar y validar el estado del agregado `Evento` (cupos disponibles) y verificar la existencia/estado del agregado `Participante`. Colocar esta lógica dentro de la entidad `Participante` violaría el principio de alta cohesión (el participante no debe manejar aforos), y colocarla en `Evento` acoplaría el agregado a flujos de identidad complejos. El servicio de dominio coordina la interacción garantizando la consistencia transaccional.

### 3. `ServicioPoliticasNotificacion` (NotificationPolicyService)
* **Responsabilidad:** Implementar las **Enterprise Business Rules** (Reglas de Negocio de la Empresa) globales que definen *cuándo*, *a quién* y *bajo qué condiciones* es obligatorio por contrato disparar una alerta (ej. umbrales de tiempo para recordatorios previos, obligatoriedad de encuestas de satisfacción según el tipo de evento, o políticas de cambios críticos de fecha).
* **Justificación de Dominio:** El enunciado exige separar la lógica a dos niveles. Mientras que enviar el email físico es un servicio de la capa de aplicación/infraestructura, **determinar si corresponde notificar según el contrato del evento** es una regla de negocio pura de la corporación. Al cambiar con alta frecuencia según el tipo de evento, encapsular esta política en un servicio de dominio protege a las entidades de la volatilidad comercial.

### 4. `ValidadorCompatibilidadServicios` (ServiceAddOnValidator)
* **Responsabilidad:** Validar de forma cruzada que los requerimientos técnicos y logísticos de los agregados externos (`Catering` y `Audiovisual`) sean viables y compatibles con la infraestructura, ubicación y tipo de `Evento` seleccionado (ej. verificar que la locación cuente con la carga eléctrica requerida por los equipos AV, o que el volumen de comida guarde coherencia con la lista de asistentes).
* **Justificación de Dominio:** Es una validación estructural de negocio que evita el registro de datos inconsistentes en el sistema. Al requerir la evaluación de las invariantes de la locación del `Evento` contra las fichas técnicas del `Catering` y el `Audiovisual`, la lógica debe centralizarse en un componente neutral sin estado.

---

## 🛠️ Matriz de Arquitectura Limpia: Separación de Reglas de Negocio

Para dar cumplimiento estricto al requerimiento de las notificaciones a doble nivel, el comportamiento se distribuye de la siguiente forma:

| Capa Arquitectónica | Componente / Patrón | Responsabilidad Específica en el Sistema |
| :--- | :--- | :--- |
| **Dominio** (Enterprise Rules) | `ServicioPoliticasNotificacion` | Evalúa las condiciones, variables del evento y determina si la notificación **debe** existir según la estrategia comercial. |
| **Aplicación** (Application Rules) | `Caso de Uso / Application Service` | Orquesta el flujo: intercepta el cambio de estado del evento, consulta al servicio de dominio si aplica notificar, mapea los datos al DTO e invoca el puerto. |
| **Infraestructura** (Adapters) | `NotificationAdapter (SMTP/SMS)` | Implementación técnica del puerto (ej. consumir la API externa de correos o Twilio para despachar el mensaje). |

---

## 📐 Lienzo de Diseño Táctico (Para Resolver en Simulacro)

### Ejercicio 1: Identificación de Value Objects (Evitar Obsesión Primitiva)
*Completa los tipos tácticos necesarios para blindar los atributos del Participante y los costos:*

| Nombre del Value Object | Atributos Componentes | Invariantes a Validar en el Constructor |
| :--- | :--- | :--- |
| `Cedula` | `value: string` | Debe cumplir el formato legal. No puede ser vacía. |
| `Genero` | `value: string` | **Restricción:** Solo permite estrictamente "Femenino" o "Masculino". |
| `Email` | `value: string` | Expresión regular de correo válida estructuralmente. |
| `Money` | `amount: decimal, currency: string` | El monto no puede ser inferior a cero. |

### Ejercicio 2: Pseudo-código del Orquestador de Notificaciones (Doble Nivel)
*Práctica cómo interactúan la Capa de Aplicación (Application Business Rules) y tu Servicio de Dominio (Enterprise Business Rules):*

```typescript
// Escribe aquí la propuesta de flujo en TypeScript / C#
// Tip UCAB: Invoca al Servicio de Dominio dentro del Caso de Uso para validar el "if (policy.ShouldNotify(...))"
````

### Ejercicio 3: Flujo de Eventos de Dominio ante Cancelaciones

Plaintext

```
[Caso de Uso: CancelarEvento]
  ├── 1. Evento AR: Muta estado a CANCELADO y dispara Evento: EventoCancelado
  ├── 2. Domain Event Listener: Intercepta el evento en la capa de Aplicación
  ├── 3. Domain Service: GestorInscripciones limpia la lista de espera de ese ID
  └── 4. Application Service: Invoca interfaces para notificar a los participantes afectados
```

```

***

### 💡 Consejo del ingeniero para el examen:
Ojo con el VO de **Género**. El enunciado te pone una restricción explícita: *"solo se permite femenino y masculino"*. La mejor manera de demostrar criterio de arquitectura en la solución en código es modelar ese Value Object utilizando un **Enum** interno o validando directamente en el constructor del objeto de valor para que lance una excepción de dominio si el string no coincide. Eso garantiza que un participante inválido jamás pueda ser instanciado en memoria.

¿Procedemos a guardar esta estructura y a subirla a tu repositorio con un `git push`?
```
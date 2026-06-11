
# 🔍 Ejercicio de Refactorización: Architectural Forensics (Caso .NET)

> [!danger] Code Smell Detectado: Big Ball of Mud & Modelo Anémico
> El siguiente componente presenta acoplamiento severo, fuga de infraestructura y un dominio anémico, violando los principios fundamentales de SOLID y Clean Architecture.

---

## 💻 Código Fuente Original (Legado)

```csharp
// LoanApplicationController.cs
[ApiController]
[Route("api/loans")]
public class LoanApplicationController : ControllerBase
{
    [HttpPost("apply")]
    public async Task<IActionResult> Apply(string customerId, decimal amount, string purpose)
    {
        var conn = new SqlConnection("Server-prod-db01; Database=LoanCore; User Id=sa;Password=Pr0d$ecure!");
        await conn.OpenAsync();

        var custCmd = new SqlCommand("SELECT * FROM Customers WHERE Id = @id", conn);
        custCmd.Parameters.AddWithValue("@id", customerId);
        using var reader = await custCmd.ExecuteReaderAsync();
        
        if (!reader.Read()) return NotFound("Cliente no encontrado");

        var creditScore = (int)reader["CreditScore"];
        var monthlyIncome = (decimal)reader["MonthlyIncome"];
        var email = (string)reader["Email"];
        reader.Close();

        // Regla de negocio: score mínimo requerido
        if (creditScore < 600)
            return BadRequest("Score crediticio insuficiente");

        // Regla de negocio: préstamo no puede superar 5x el ingreso mensual
        if (amount > monthlyIncome * 5)
            return BadRequest("Monto supera el límite permitido");

        // Regla de negocio: no puede tener préstamos en mora
        var delinqCmd = new SqlCommand("SELECT COUNT(*) FROM Loans WHERE CustomerId=@id AND Status='DELINQUENT'", conn);
        delinqCmd.Parameters.AddWithValue("@id", customerId);
        var delinqCount = (int)(await delinqCmd.ExecuteScalarAsync() ?? 0);
        
        if (delinqCount > 0)
            return BadRequest("Cliente con préstamos en mora");

        var insertCmd = new SqlCommand(
            "INSERT INTO Loans (CustomerId, Amount, Purpose, Status, CreatedAt) " +
            "VALUES (@cid, @amt, @pur, 'PENDING', GETDATE())", conn);
        insertCmd.Parameters.AddWithValue("@cid", customerId);
        insertCmd.Parameters.AddWithValue("@amt", amount);
        insertCmd.Parameters.AddWithValue("@pur", purpose);
        await insertCmd.ExecuteNonQueryAsync();

        var smtp = new SmtpClient("smtp.loancore.com", 587)
        {
            Credentials = new NetworkCredential("noreply@loancore.com", "Sm7p$Pass!"),
            EnableSsl = true
        };

        var mail = new MailMessage("noreply@loancore.com", email, "Solicitud recibida", "Su solicitud de préstamo está en evaluación.");
        await smtp.SendMailAsync(mail);

        return Ok("Solicitud enviada correctamente");
    }
}
````

## 🔍 Diagnóstico de Ingeniería (Code Smells & Violaciones)

### 1. Fuga de Infraestructura y Violación de Capas

El controlador (Capa de Presentación) tiene conocimiento directo de los mecanismos de persistencia (SQL) y protocolos de red (SMTP).

- **Problema:** Instanciar `SqlConnection` y `SmtpClient` directamente acopla el código a un proveedor de base de datos específico y a un servidor de correos físico.
- **Solución:** Aplicar el Principio de Inversión de Dependencias (DIP). El controlador debe delegar a un **Caso de Uso** (Aplicación), e infraestructura implementará interfaces (Puertos).

### 2. Violación del Principio de Responsabilidad Única (SRP)

El componente tiene múltiples ejes de cambio (razones para modificar el código):

1. Modificaciones en el contrato o rutas de la API HTTP.
2. Cambios en el esquema o tablas de la base de datos relacional.
3. Modificaciones en las políticas o umbrales de las reglas de negocio.
4. Cambios de proveedor de mensajería (Email, SMS, etc.).
5. Rotación o actualización de credenciales de seguridad.

### 3. Obsesión Primitiva e Invariantes no Protegidas

Conceptos de dominio críticos como montos, puntajes y correos se manejan de forma primitiva (`decimal`, `string`, `int`).

- **Problema:** Las validaciones de negocio están "sueltas" en el controlador, provocando un **Modelo de Dominio Anémico**.
- **Solución:** Extraer e instanciar _Value Objects_ (`Money`, `EmailAddress`, `CreditScore`) que protejan estructuralmente sus invariantes en el constructor.

### 4. Ausencia de Capa Anticorrupción (ACL) y Brechas de Seguridad

- **Problema:** Credenciales en texto plano (`sa`, `passwords`) grabadas a fuego en el código fuente. Adicionalmente, las filas puras del `SqlDataReader` se exponen directamente sin un mapeo intermedio.
- **Solución:** Extraer parámetros a variables de entorno y construir un Adaptador de Persistencia que transforme las consultas en Entidades de Dominio puras.

## 🛠️ Plan de Refactorización (Diseño del Target Architecture)

### Mapeo de Responsabilidades por Capa

|**Componente Original / Lógica**|**Patrón DDD / Clean Arq.**|**Capa de Destino**|
|---|---|---|
|Consultas SQL / Clientes SMTP|Adaptadores (`Repository`, `EmailSender`)|**Infraestructura**|
|Reglas de Validación (`if`)|Entidad `LoanApplication` y _Value Objects_|**Dominio**|
|Orquestación de Pasos y Flujo|Caso de Uso (`ApplyForLoanUseCase`)|**Aplicación**|
|Endpoint HTTP y Enrutamiento|`LoanApplicationController` (Filtro/Delegación)|**Presentación**|

## 📐 Lienzo de Diseño Táctico (Para Resolver)

_Utiliza este espacio en blanco para modelar los componentes limpios extraídos del ejercicio anterior:_

### Value Objects a Diseñar

|**Nombre del Value Object**|**Atributos**|**Invariantes a Validar (Constructor)**|
|---|---|---|
||||
||||

### Agregados y Entidades

|**Raíz de Agregado / Entidad**|**Identificador (ID)**|**Métodos de Negocio (Comportamiento)**|
|---|---|---|
||||

### Servicios de Dominio / Interfaces de Aplicación (Puertos)

|**Nombre de la Interfaz / Servicio**|**Métodos Clave**|**Propósito**|
|---|---|---|
||||

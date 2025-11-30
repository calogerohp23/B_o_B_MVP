# Diagrama ER y Arquitectura - MVP Banco Digital

**Documento:** Diagrama ER + Inicio de la Arquitectura (Clean Architecture)

---

## 1. Diagrama ER (Entidad-Relación)

Se incluye un diagrama compatible con Mermaid y una descripción textual para referencia rápida.

```mermaid
erDiagram
    CUSTOMERS {
        UUID CustomerId PK
        NVARCHAR FullName
        NVARCHAR Email
        NVARCHAR Phone
        NVARCHAR DocumentNumber
        DATETIME CreatedAt
    }

    ACCOUNTS {
        UUID AccountId PK
        UUID CustomerId FK
        VARCHAR AccountNumber
        VARCHAR AccountType
        DECIMAL Balance
        VARCHAR Currency
        VARCHAR Status
        DATETIME CreatedAt
    }

    TRANSACTIONS {
        UUID TransactionId PK
        UUID AccountId FK
        VARCHAR Type
        DECIMAL Amount
        NVARCHAR Description
        DATETIME CreatedAt
        UUID RelatedTransferId FK NULL
        UUID PerformedByUserId FK NULL
    }

    TRANSFERS {
        UUID TransferId PK
        UUID FromAccountId FK
        UUID ToAccountId FK
        DECIMAL Amount
        DATETIME CreatedAt
        NVARCHAR Status
    }

    USERS {
        UUID UserId PK
        NVARCHAR Username
        VARBINARY PasswordHash
        VARCHAR Role
        DATETIME CreatedAt
    }

    AUDITLOGS {
        UUID AuditId PK
        UUID UserId FK NULL
        NVARCHAR Action
        NVARCHAR Entity
        NVARCHAR EntityId
        DATETIME Timestamp
        NVARCHAR Details NULL
    }

    CUSTOMERS ||--o{ ACCOUNTS : "tiene"
    ACCOUNTS ||--o{ TRANSACTIONS : "registra"
    ACCOUNTS ||--o{ TRANSFERS : "origen"
    ACCOUNTS ||--o{ TRANSFERS : "destino"
    TRANSFERS ||--o{ TRANSACTIONS : "genera"
    USERS ||--o{ AUDITLOGS : "genera"
    USERS ||--o{ TRANSACTIONS : "realiza"
```

### Notas de diseño del ER
- **Customers - Accounts:** relación 1:N (un cliente puede tener varias cuentas).
- **Accounts - Transactions:** relación 1:N.
- **Transfers:** entidad representando la operación de transferencia. Cada transferencia generará al menos dos transacciones (TransferOut y TransferIn) que referencian `RelatedTransferId`.
- **Users:** usuarios del sistema (administradores, cajeros, auditores). Pueden estar separados de Customer porque en el MVP no habrá acceso del cliente final.
- **AuditLogs:** registros inmarchables para auditoría.

---

## 2. Modelo lógico (DDL simplificado)

Proporciono un DDL básico (SQL Server) que puede usarse como punto de partida.

```sql
CREATE TABLE Customers (
    CustomerId UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    FullName NVARCHAR(150) NOT NULL,
    Email NVARCHAR(150) UNIQUE NOT NULL,
    Phone NVARCHAR(20),
    DocumentNumber NVARCHAR(30) UNIQUE,
    CreatedAt DATETIME2 DEFAULT SYSDATETIME()
);

CREATE TABLE Accounts (
    AccountId UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    CustomerId UNIQUEIDENTIFIER NOT NULL,
    AccountNumber VARCHAR(20) UNIQUE NOT NULL,
    AccountType VARCHAR(30) NOT NULL,
    Balance DECIMAL(18,2) NOT NULL DEFAULT 0,
    Currency VARCHAR(10) NOT NULL DEFAULT 'DOP',
    Status VARCHAR(20) NOT NULL DEFAULT 'Active',
    CreatedAt DATETIME2 DEFAULT SYSDATETIME(),
    CONSTRAINT FK_Accounts_Customers FOREIGN KEY (CustomerId) REFERENCES Customers(CustomerId)
);

CREATE TABLE Users (
    UserId UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    Username NVARCHAR(50) UNIQUE NOT NULL,
    PasswordHash VARBINARY(MAX) NOT NULL,
    Role VARCHAR(50) NOT NULL,
    CreatedAt DATETIME2 DEFAULT SYSDATETIME()
);

CREATE TABLE Transfers (
    TransferId UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    FromAccountId UNIQUEIDENTIFIER NOT NULL,
    ToAccountId UNIQUEIDENTIFIER NOT NULL,
    Amount DECIMAL(18,2) NOT NULL,
    Status VARCHAR(20) NOT NULL DEFAULT 'Completed',
    CreatedAt DATETIME2 DEFAULT SYSDATETIME(),
    CONSTRAINT FK_Transfers_FromAccount FOREIGN KEY (FromAccountId) REFERENCES Accounts(AccountId),
    CONSTRAINT FK_Transfers_ToAccount FOREIGN KEY (ToAccountId) REFERENCES Accounts(AccountId)
);

CREATE TABLE Transactions (
    TransactionId UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    AccountId UNIQUEIDENTIFIER NOT NULL,
    Type VARCHAR(20) NOT NULL,
    Amount DECIMAL(18,2) NOT NULL,
    Description NVARCHAR(200),
    RelatedTransferId UNIQUEIDENTIFIER NULL,
    PerformedByUserId UNIQUEIDENTIFIER NULL,
    CreatedAt DATETIME2 DEFAULT SYSDATETIME(),
    CONSTRAINT FK_Transactions_Accounts FOREIGN KEY (AccountId) REFERENCES Accounts(AccountId),
    CONSTRAINT FK_Transactions_Transfers FOREIGN KEY (RelatedTransferId) REFERENCES Transfers(TransferId),
    CONSTRAINT FK_Transactions_Users FOREIGN KEY (PerformedByUserId) REFERENCES Users(UserId)
);

CREATE TABLE AuditLogs (
    AuditId UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    UserId UNIQUEIDENTIFIER NULL,
    Action NVARCHAR(200),
    Entity NVARCHAR(100),
    EntityId NVARCHAR(100),
    Details NVARCHAR(MAX) NULL,
    Timestamp DATETIME2 DEFAULT SYSDATETIME(),
    CONSTRAINT FK_AuditLogs_Users FOREIGN KEY (UserId) REFERENCES Users(UserId)
);
```

---

## 3. Índices y consideraciones de rendimiento
- Indices sugeridos:
  - `Accounts(AccountNumber) UNIQUE` (ya implícito)
  - `Transactions(AccountId, CreatedAt)` para consultas por cuenta y rango.
  - `Transfers(FromAccountId, ToAccountId, CreatedAt)` si hay consultas por origen/destino.
  - `AuditLogs(Timestamp)` para búsquedas por rango.
- Particionamiento: para el MVP no es necesario, pero planificar particionamiento por fecha para `Transactions` cuando el volumen crezca.
- Uso de `DECIMAL(18,2)` para montos: suficiente para dinero en la mayoría de mercados.

---

## 4. Iniciando la Arquitectura (Clean Architecture) — Visión general
A continuación comienzo a definir la estructura del proyecto, sus responsabilidades y ejemplos de código para las capas principales.

### Estructura de carpetas propuesta

```
/src
  /Banking.Domain
    /Entities
    /ValueObjects
    /Exceptions
    /Interfaces
  /Banking.Application
    /DTOs
    /Commands
    /Queries
    /Handlers
    /Services
  /Banking.Infrastructure
    /Persistence
      /Migrations
      AppDbContext.cs
    /Repositories
    /Security
  /Banking.API
    /Controllers
    /Models
    /Mappings
    Program.cs
/tests
  /Banking.UnitTests
  /Banking.IntegrationTests
```

### Responsabilidades por capa
- **Domain:** Reglas de negocio puras, entidades con comportamientos (ej: `Account.Deposit()`). No referencias a frameworks.
- **Application:** Casos de uso (Commands/Queries), DTOs, interfaces de repositorio. Aquí se define `IAccountRepository`, `IUnitOfWork`, y los handlers usando MediatR o un patrón similar.
- **Infrastructure:** Implementaciones concretas (EF Core `AppDbContext`, repositorios, hashing de contraseñas, logger). Dependencias externas.
- **API:** Web API minimalista que orquesta requests -> Application (MediatR) -> Domain.

---

## 5. Ejemplos de código (arranque rápido)

### 5.1 Entidad de dominio: Account (C#)

```csharp
namespace Banking.Domain.Entities
{
    public class Account
    {
        public Guid AccountId { get; private set; }
        public Guid CustomerId { get; private set; }
        public string AccountNumber { get; private set; }
        public decimal Balance { get; private set; }
        public string Currency { get; private set; }
        public string Status { get; private set; }

        private Account() { }

        public static Account Create(Guid customerId, string accountNumber, string accountType, string currency = "DOP")
        {
            return new Account
            {
                AccountId = Guid.NewGuid(),
                CustomerId = customerId,
                AccountNumber = accountNumber,
                Balance = 0m,
                Currency = currency,
                Status = "Active"
            };
        }

        public void Deposit(decimal amount)
        {
            if (amount <= 0) throw new ArgumentException("El monto debe ser positivo", nameof(amount));
            Balance += amount;
        }

        public void Withdraw(decimal amount)
        {
            if (amount <= 0) throw new ArgumentException("El monto debe ser positivo", nameof(amount));
            if (Balance < amount) throw new InvalidOperationException("Fondos insuficientes");
            Balance -= amount;
        }

        public void Block(string reason)
        {
            Status = "Blocked";
            // podría registrar un DomainEvent
        }
    }
}
```

### 5.2 Interfaz de repositorio (Application/Domain)

```csharp
public interface IAccountRepository
{
    Task<Account> GetByIdAsync(Guid accountId);
    Task<Account> GetByAccountNumberAsync(string accountNumber);
    Task AddAsync(Account account);
    Task UpdateAsync(Account account);
}
```

### 5.3 AppDbContext (Infrastructure/Persistence)

```csharp
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Customer> Customers { get; set; }
    public DbSet<Account> Accounts { get; set; }
    public DbSet<Transaction> Transactions { get; set; }
    public DbSet<Transfer> Transfers { get; set; }
    public DbSet<User> Users { get; set; }
    public DbSet<AuditLog> AuditLogs { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
        base.OnModelCreating(modelBuilder);
    }
}
```

### 5.4 Ejemplo de Handler para Transfer (Application/Handlers)

```csharp
public class TransferCommand : IRequest<Result>
{
    public Guid FromAccountId { get; set; }
    public Guid ToAccountId { get; set; }
    public decimal Amount { get; set; }
    public Guid PerformedByUserId { get; set; }
}

public class TransferCommandHandler : IRequestHandler<TransferCommand, Result>
{
    private readonly IAccountRepository _accountRepo;
    private readonly ITransferRepository _transferRepo;
    private readonly ITransactionRepository _transactionRepo;
    private readonly IUnitOfWork _uow;

    public async Task<Result> Handle(TransferCommand request, CancellationToken cancellationToken)
    {
        var from = await _accountRepo.GetByIdAsync(request.FromAccountId);
        var to = await _accountRepo.GetByIdAsync(request.ToAccountId);

        if (from == null || to == null) return Result.Failure("Cuenta no encontrada");
        if (from.Status != "Active") return Result.Failure("Cuenta origen no activa");
        if (to.Status != "Active") return Result.Failure("Cuenta destino no activa");
        if (from.Balance < request.Amount) return Result.Failure("Fondos insuficientes");

        // Operación dentro de una unidad de trabajo
        from.Withdraw(request.Amount);
        to.Deposit(request.Amount);

        var transfer = Transfer.Create(from.AccountId, to.AccountId, request.Amount);
        await _transferRepo.AddAsync(transfer);

        var txOut = Transaction.Create(from.AccountId, "TransferOut", request.Amount, null, transfer.TransferId, request.PerformedByUserId);
        var txIn  = Transaction.Create(to.AccountId, "TransferIn", request.Amount, null, transfer.TransferId, request.PerformedByUserId);

        await _transactionRepo.AddAsync(txOut);
        await _transactionRepo.AddAsync(txIn);

        await _accountRepo.UpdateAsync(from);
        await _accountRepo.UpdateAsync(to);

        await _uow.SaveChangesAsync();

        return Result.Success();
    }
}
```

---

## 6. Siguientes pasos que ya incluí en el documento (y que puedo generar ahora mismo)
- Generar proyecto base en C# con la estructura de carpetas y archivos mostrados.
- Crear migraciones EF Core y script inicial de la base de datos.
- Implementar tests unitarios para `Account` y `TransferCommandHandler`.
- Generar diagramas UML (clases y secuencia) para transferencias.

---

**Fin del documento.**


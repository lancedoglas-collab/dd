# GridShield 2026 — Full-Stack Implementation Plan
**Topic:** Renewable Energy Distribution Network — San Nicolas Smart Grid (CEDC)
**Stack:** React + Vite (frontend) · ASP.NET Core Web API (backend) · Supabase PostgreSQL (database)
**Deadline:** This week (April 14–18, 2026)

---

## Selected 5 Modules

| # | Module | Topic Relevance |
|---|--------|-----------------|
| 1 | **Authentication & Login** | Role-gated access for grid operators, engineers, analysts |
| 2 | **Dashboard** | Live KPI overview — load, endpoints, grid health |
| 3 | **SCADA Monitor** | Voltage/Load/Frequency per RTU + Solar Node capacity utilization ← core of topic |
| 4 | **Incident Management** | Grid fault and anomaly tracking — full CRUD |
| 5 | **User Management** | Admin CRUD for all grid personnel roles |

---

## Project Structure

```
gridshield2026/
├── frontend/                        ← React + Vite (trimmed to 5 modules)
│   ├── src/
│   │   ├── api/
│   │   │   ├── client.js            ← Axios instance + JWT interceptor
│   │   │   ├── auth.js
│   │   │   ├── dashboard.js         ← stats + SignalR telemetry
│   │   │   ├── scada.js             ← RTU telemetry + solar nodes
│   │   │   ├── incidents.js
│   │   │   └── users.js
│   │   └── App.jsx                  ← Trimmed to 5 modules
│   └── package.json
│
└── backend/
    ├── GridShield.sln
    ├── GridShield.API/              ← ASP.NET Core Web API
    │   ├── Controllers/
    │   │   ├── AuthController.cs
    │   │   ├── DashboardController.cs
    │   │   ├── ScadaController.cs
    │   │   ├── IncidentsController.cs
    │   │   └── UsersController.cs
    │   ├── Hubs/
    │   │   └── GridHub.cs           ← SignalR real-time hub
    │   ├── Middleware/
    │   │   └── ExceptionMiddleware.cs
    │   ├── appsettings.json
    │   └── Program.cs
    ├── GridShield.Core/             ← Domain layer (no dependencies)
    │   ├── Entities/
    │   │   ├── User.cs
    │   │   ├── Incident.cs
    │   │   ├── RtuTelemetry.cs
    │   │   └── SolarNode.cs
    │   ├── DTOs/
    │   │   ├── AuthDtos.cs
    │   │   ├── IncidentDtos.cs
    │   │   ├── UserDtos.cs
    │   │   └── ScadaDtos.cs
    │   ├── Interfaces/
    │   │   ├── IRepository.cs
    │   │   ├── IIncidentRepository.cs
    │   │   ├── IUserRepository.cs
    │   │   ├── IScadaRepository.cs
    │   │   └── IAuthService.cs
    │   └── Features/                ← CQRS Commands & Queries (MediatR)
    │       ├── Incidents/
    │       │   ├── Commands/
    │       │   └── Queries/
    │       └── Users/
    │           ├── Commands/
    │           └── Queries/
    ├── GridShield.Infrastructure/   ← EF Core + Supabase PostgreSQL
    │   ├── Data/
    │   │   └── GridShieldDbContext.cs
    │   ├── Repositories/
    │   │   ├── IncidentRepository.cs
    │   │   ├── UserRepository.cs
    │   │   └── ScadaRepository.cs
    │   └── Services/
    │       ├── AuthService.cs
    │       └── TelemetryBroadcaster.cs  ← BackgroundService (SignalR push every 2s)
    └── GridShield.Tests/
        └── IncidentHandlerTests.cs
```

---

## NuGet Packages

```bash
# GridShield.API
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package Microsoft.AspNetCore.SignalR
dotnet add package Swashbuckle.AspNetCore
dotnet add package Serilog.AspNetCore
dotnet add package AutoMapper.Extensions.Microsoft.DependencyInjection
dotnet add package FluentValidation.AspNetCore

# GridShield.Infrastructure
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package BCrypt.Net-Next
dotnet add package MediatR.Extensions.Microsoft.DependencyInjection

# GridShield.Tests
dotnet add package Moq
dotnet add package FluentAssertions
```

---

## Database (Supabase + EF Core / Npgsql)

### appsettings.json
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=db.<ref>.supabase.co;Port=5432;Database=postgres;Username=postgres;Password=<password>;SSL Mode=Require;Trust Server Certificate=true"
  },
  "Jwt": {
    "Secret": "<min-32-char-secret>",
    "Issuer": "gridshield-api",
    "Audience": "gridshield-client"
  }
}
```

---

## Domain Entities

### User.cs
```csharp
public class User
{
    public Guid Id { get; set; }
    public string Username { get; set; } = string.Empty;
    public string PasswordHash { get; set; } = string.Empty;
    public string Name { get; set; } = string.Empty;
    public UserRole Role { get; set; }
    public bool IsActive { get; set; } = true;
    public string ZoneAccess { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }
}

public enum UserRole { Administrator, GridOperator, SecurityAnalyst, FieldEngineer }
```

### Incident.cs
```csharp
public class Incident
{
    public Guid Id { get; set; }
    public string Title { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
    public IncidentSeverity Severity { get; set; }
    public IncidentStatus Status { get; set; }
    public string AffectedZone { get; set; } = string.Empty;
    public string AssignedTo { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }
    public DateTime? ResolvedAt { get; set; }
    public Guid CreatedByUserId { get; set; }
}

public enum IncidentSeverity { Critical, High, Medium, Low }
public enum IncidentStatus   { Open, InProgress, Resolved, Closed }
```

### RtuTelemetry.cs  ← NEW (Voltage Management)
```csharp
public class RtuTelemetry
{
    public Guid Id { get; set; }
    public string RtuId { get; set; } = string.Empty;       // "RTU-01"
    public string Location { get; set; } = string.Empty;    // "San Nicolas North"
    public double VoltageKv { get; set; }                   // e.g. 13.841
    public double LoadMw { get; set; }                      // e.g. 4.21
    public double FrequencyHz { get; set; }                 // e.g. 59.983
    public string Status { get; set; } = "ONLINE";
    public string PqcCipher { get; set; } = "Kyber-1024";
    public DateTime RecordedAt { get; set; }
}
```

### SolarNode.cs  ← NEW (Renewable Energy)
```csharp
public class SolarNode
{
    public Guid Id { get; set; }
    public string NodeId { get; set; } = string.Empty;      // "Solar Node 01"
    public double CapacityUtilizationPct { get; set; }      // 0–100
    public bool IsOnline { get; set; } = true;
    public double OutputKw { get; set; }
    public DateTime LastUpdated { get; set; }
}
```

---

## DbContext

```csharp
public class GridShieldDbContext(DbContextOptions<GridShieldDbContext> options)
    : DbContext(options)
{
    public DbSet<User>         Users        { get; set; }
    public DbSet<Incident>     Incidents    { get; set; }
    public DbSet<RtuTelemetry> RtuTelemetry { get; set; }
    public DbSet<SolarNode>    SolarNodes   { get; set; }

    protected override void OnModelCreating(ModelBuilder mb)
    {
        // PostgreSQL native UUID generation
        mb.Entity<User>().Property(u => u.Id).HasDefaultValueSql("gen_random_uuid()");
        mb.Entity<Incident>().Property(i => i.Id).HasDefaultValueSql("gen_random_uuid()");
        mb.Entity<RtuTelemetry>().Property(r => r.Id).HasDefaultValueSql("gen_random_uuid()");
        mb.Entity<SolarNode>().Property(s => s.Id).HasDefaultValueSql("gen_random_uuid()");

        mb.Entity<User>().HasIndex(u => u.Username).IsUnique();

        // Seed users
        mb.Entity<User>().HasData(
            new User { Id = Guid.Parse("00000000-0000-0000-0000-000000000001"),
                       Username = "admin",    Name = "System Administrator",
                       PasswordHash = BCrypt.Net.BCrypt.HashPassword("admin123"),
                       Role = UserRole.Administrator, IsActive = true,
                       ZoneAccess = "All", CreatedAt = DateTime.UtcNow },
            new User { Id = Guid.Parse("00000000-0000-0000-0000-000000000002"),
                       Username = "operator", Name = "CEDC Operator",
                       PasswordHash = BCrypt.Net.BCrypt.HashPassword("ops2026"),
                       Role = UserRole.GridOperator, IsActive = true,
                       ZoneAccess = "Zone-1,Zone-2", CreatedAt = DateTime.UtcNow },
            new User { Id = Guid.Parse("00000000-0000-0000-0000-000000000003"),
                       Username = "analyst",  Name = "SOC Analyst",
                       PasswordHash = BCrypt.Net.BCrypt.HashPassword("analyst26"),
                       Role = UserRole.SecurityAnalyst, IsActive = true,
                       ZoneAccess = "Zone-1,Zone-3", CreatedAt = DateTime.UtcNow },
            new User { Id = Guid.Parse("00000000-0000-0000-0000-000000000004"),
                       Username = "engineer", Name = "Substation Engineer",
                       PasswordHash = BCrypt.Net.BCrypt.HashPassword("eng2026"),
                       Role = UserRole.FieldEngineer, IsActive = true,
                       ZoneAccess = "Zone-2", CreatedAt = DateTime.UtcNow }
        );

        // Seed RTU stations
        mb.Entity<RtuTelemetry>().HasData(
            new RtuTelemetry { Id = Guid.Parse("00000000-0000-0000-0001-000000000001"),
                               RtuId = "RTU-01", Location = "San Nicolas North",
                               VoltageKv = 13.841, LoadMw = 4.21,
                               FrequencyHz = 59.983, RecordedAt = DateTime.UtcNow },
            new RtuTelemetry { Id = Guid.Parse("00000000-0000-0000-0001-000000000002"),
                               RtuId = "RTU-02", Location = "San Nicolas Central",
                               VoltageKv = 13.792, LoadMw = 6.14,
                               FrequencyHz = 60.011, RecordedAt = DateTime.UtcNow },
            new RtuTelemetry { Id = Guid.Parse("00000000-0000-0000-0001-000000000003"),
                               RtuId = "RTU-03", Location = "San Nicolas Port",
                               VoltageKv = 13.810, LoadMw = 5.72,
                               FrequencyHz = 59.990, RecordedAt = DateTime.UtcNow }
        );

        // Seed solar nodes
        mb.Entity<SolarNode>().HasData(
            Enumerable.Range(1, 6).Select(i => new SolarNode
            {
                Id = Guid.Parse($"00000000-0000-0000-0002-{i:D12}"),
                NodeId = $"Solar Node {i:D2}",
                CapacityUtilizationPct = 40 + (i * 7 % 55),
                OutputKw = 120 + (i * 15),
                IsOnline = true,
                LastUpdated = DateTime.UtcNow
            }).ToArray()
        );
    }
}
```

### Migrations

```bash
dotnet ef migrations add InitialCreate \
  --project GridShield.Infrastructure \
  --startup-project GridShield.API

dotnet ef database update \
  --project GridShield.Infrastructure \
  --startup-project GridShield.API
```

---

## Backend — Key Implementations

### Interfaces

```csharp
// IRepository.cs — generic base
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<IEnumerable<T>> GetAllAsync(CancellationToken ct = default);
    Task AddAsync(T entity, CancellationToken ct = default);
    Task UpdateAsync(T entity, CancellationToken ct = default);
    Task DeleteAsync(Guid id, CancellationToken ct = default);
}

// IScadaRepository.cs
public interface IScadaRepository
{
    Task<IEnumerable<RtuTelemetry>> GetLatestReadingsAsync(CancellationToken ct = default);
    Task RecordReadingAsync(RtuTelemetry reading, CancellationToken ct = default);
    Task<IEnumerable<SolarNode>> GetSolarNodesAsync(CancellationToken ct = default);
    Task UpdateSolarNodeAsync(SolarNode node, CancellationToken ct = default);
}

// IIncidentRepository.cs
public interface IIncidentRepository : IRepository<Incident>
{
    Task<IEnumerable<Incident>> GetByStatusAsync(IncidentStatus status, CancellationToken ct = default);
    Task<IEnumerable<Incident>> GetBySeverityAsync(IncidentSeverity severity, CancellationToken ct = default);
}

// IUserRepository.cs
public interface IUserRepository : IRepository<User>
{
    Task<User?> GetByUsernameAsync(string username, CancellationToken ct = default);
    Task<bool> UsernameExistsAsync(string username, CancellationToken ct = default);
}
```

### CQRS with MediatR — Incident Example

```csharp
// CreateIncidentCommand.cs
public record CreateIncidentCommand(
    string Title, string Description,
    IncidentSeverity Severity, string AffectedZone,
    string AssignedTo, Guid CreatedByUserId
) : IRequest<IncidentDto>;

// CreateIncidentHandler.cs
public class CreateIncidentHandler(
    IIncidentRepository repo,
    IMapper mapper
) : IRequestHandler<CreateIncidentCommand, IncidentDto>
{
    public async Task<IncidentDto> Handle(CreateIncidentCommand cmd, CancellationToken ct)
    {
        var incident = mapper.Map<Incident>(cmd);
        incident.Id = Guid.NewGuid();
        incident.CreatedAt = DateTime.UtcNow;
        incident.Status = IncidentStatus.Open;

        await repo.AddAsync(incident, ct);
        return mapper.Map<IncidentDto>(incident);
    }
}
```

### JWT Auth Service

```csharp
public class AuthService(IUserRepository userRepo, IConfiguration config) : IAuthService
{
    public async Task<AuthResult> LoginAsync(LoginRequest request)
    {
        var user = await userRepo.GetByUsernameAsync(request.Username);

        if (user is null || !BCrypt.Net.BCrypt.Verify(request.Password, user.PasswordHash))
            return AuthResult.Failure("Invalid credentials");

        return AuthResult.Success(GenerateJwt(user), user);
    }

    private string GenerateJwt(User user)
    {
        var key   = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(config["Jwt:Secret"]!));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
            new Claim(ClaimTypes.Name, user.Username),
            new Claim(ClaimTypes.Role, user.Role.ToString()),
            new Claim("FullName", user.Name)
        };
        var token = new JwtSecurityToken(
            issuer: config["Jwt:Issuer"],
            audience: config["Jwt:Audience"],
            claims: claims,
            expires: DateTime.UtcNow.AddHours(8),
            signingCredentials: creds
        );
        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

### SignalR Hub + Background Telemetry Broadcaster

```csharp
// GridHub.cs
[Authorize]
public class GridHub : Hub
{
    public async Task JoinDashboard() =>
        await Groups.AddToGroupAsync(Context.ConnectionId, "dashboard");

    public async Task JoinScada() =>
        await Groups.AddToGroupAsync(Context.ConnectionId, "scada");
}

// TelemetryBroadcaster.cs — pushes live RTU + solar data every 2s
public class TelemetryBroadcaster(
    IHubContext<GridHub> hub,
    IServiceScopeFactory scopeFactory
) : BackgroundService
{
    private readonly double[] _baseVoltage   = [13.84, 13.79, 13.81];
    private readonly double[] _baseLoad      = [4.2,   6.1,   5.7 ];
    private readonly double[] _baseFrequency = [59.98, 60.01, 59.99];
    private readonly string[] _rtuIds        = ["RTU-01", "RTU-02", "RTU-03"];
    private readonly string[] _locations     = ["San Nicolas North", "San Nicolas Central", "San Nicolas Port"];

    private int _tick = 0;

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            _tick++;

            // Build live RTU readings using sine wave (mirrors the React simulation)
            var rtus = Enumerable.Range(0, 3).Select(i => new
            {
                rtuId     = _rtuIds[i],
                location  = _locations[i],
                voltageKv = Math.Round(_baseVoltage[i]   + Math.Sin(_tick * 0.3) * 0.02, 3),
                loadMw    = Math.Round(_baseLoad[i]      + Math.Sin(_tick * 0.3) * 0.3,  2),
                frequencyHz = Math.Round(_baseFrequency[i] + Math.Sin(_tick * 0.1) * 0.03, 3),
                status    = "ONLINE",
                timestamp = DateTime.UtcNow
            }).ToList();

            // Build live solar node readings
            var solarNodes = Enumerable.Range(1, 6).Select(i => new
            {
                nodeId = $"Solar Node {i:D2}",
                capacityPct = Math.Max(10, Math.Min(95,
                    40 + (int)(Math.Sin((_tick + i) * 0.5) * 18))),
                isOnline = true
            }).ToList();

            // Push to SCADA group
            await hub.Clients.Group("scada").SendAsync("scadaUpdate", new
            {
                rtus,
                solarNodes,
                timestamp = DateTime.UtcNow
            }, ct);

            // Push summary to dashboard group
            await hub.Clients.Group("dashboard").SendAsync("dashboardUpdate", new
            {
                totalLoadMw = rtus.Sum(r => r.loadMw),
                avgFrequencyHz = rtus.Average(r => r.frequencyHz),
                avgSolarPct = solarNodes.Average(n => n.capacityPct),
                timestamp = DateTime.UtcNow
            }, ct);

            await Task.Delay(2000, ct);
        }
    }
}
```

### ScadaController.cs ← NEW

```csharp
[ApiController]
[Route("api/scada")]
[Authorize]
public class ScadaController(IScadaRepository repo) : ControllerBase
{
    // GET api/scada/telemetry — latest reading per RTU
    [HttpGet("telemetry")]
    public async Task<IActionResult> GetTelemetry(CancellationToken ct) =>
        Ok(await repo.GetLatestReadingsAsync(ct));

    // GET api/scada/solar-nodes — all solar nodes
    [HttpGet("solar-nodes")]
    public async Task<IActionResult> GetSolarNodes(CancellationToken ct) =>
        Ok(await repo.GetSolarNodesAsync(ct));

    // POST api/scada/telemetry — record a new RTU reading (grid operators only)
    [HttpPost("telemetry")]
    [Authorize(Policy = "CanManageGrid")]
    public async Task<IActionResult> RecordReading(RtuTelemetry reading, CancellationToken ct)
    {
        reading.Id = Guid.NewGuid();
        reading.RecordedAt = DateTime.UtcNow;
        await repo.RecordReadingAsync(reading, ct);
        return CreatedAtAction(nameof(GetTelemetry), reading);
    }

    // PATCH api/scada/solar-nodes/{id} — update solar node output
    [HttpPatch("solar-nodes/{id:guid}")]
    [Authorize(Policy = "CanManageGrid")]
    public async Task<IActionResult> UpdateSolarNode(Guid id, SolarNode node, CancellationToken ct)
    {
        node.Id = id;
        node.LastUpdated = DateTime.UtcNow;
        await repo.UpdateSolarNodeAsync(node, ct);
        return NoContent();
    }
}
```

### Global Exception Middleware

```csharp
public class ExceptionMiddleware(RequestDelegate next, ILogger<ExceptionMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext ctx)
    {
        try { await next(ctx); }
        catch (Exception ex)
        {
            logger.LogError(ex, "Unhandled exception at {Path}", ctx.Request.Path);
            ctx.Response.StatusCode = 500;
            await ctx.Response.WriteAsJsonAsync(new { error = "An internal error occurred." });
        }
    }
}
```

### Authorization Policies

```csharp
builder.Services.AddAuthorization(opts =>
{
    opts.AddPolicy("CanManageUsers",
        p => p.RequireRole("Administrator"));

    opts.AddPolicy("CanManageIncidents",
        p => p.RequireRole("Administrator", "GridOperator", "SecurityAnalyst"));

    opts.AddPolicy("CanManageGrid",
        p => p.RequireRole("Administrator", "GridOperator"));

    opts.AddPolicy("ReadOnly",
        p => p.RequireAuthenticatedUser());
});
```

### CORS for React Dev Server

```csharp
builder.Services.AddCors(o => o.AddPolicy("ReactApp", p =>
    p.WithOrigins("http://localhost:5173")
     .AllowAnyHeader()
     .AllowAnyMethod()
     .AllowCredentials()));   // required for SignalR

app.UseCors("ReactApp");
```

---

## API Endpoints

| Method | Route | Auth Policy | Description |
|--------|-------|-------------|-------------|
| POST | `/api/auth/login` | Public | Returns JWT token |
| POST | `/api/auth/logout` | Bearer | Logs logout |
| GET | `/api/dashboard/stats` | ReadOnly | KPI summary counts |
| GET | `/api/scada/telemetry` | ReadOnly | Latest RTU readings (voltage, load, freq) |
| POST | `/api/scada/telemetry` | CanManageGrid | Record new RTU reading |
| GET | `/api/scada/solar-nodes` | ReadOnly | All 6 solar node statuses |
| PATCH | `/api/scada/solar-nodes/{id}` | CanManageGrid | Update solar node output |
| GET | `/api/incidents` | ReadOnly | List with optional `?status=` filter |
| GET | `/api/incidents/{id}` | ReadOnly | Single incident detail |
| POST | `/api/incidents` | CanManageIncidents | Create new incident |
| PUT | `/api/incidents/{id}` | CanManageIncidents | Full update |
| PATCH | `/api/incidents/{id}/status` | CanManageIncidents | Status-only update |
| DELETE | `/api/incidents/{id}` | CanManageUsers | Delete incident |
| GET | `/api/users` | CanManageUsers | List all users |
| POST | `/api/users` | CanManageUsers | Add new user |
| PUT | `/api/users/{id}` | CanManageUsers | Update user |
| PATCH | `/api/users/{id}/toggle-active` | CanManageUsers | Toggle active flag |
| DELETE | `/api/users/{id}` | CanManageUsers | Delete user |

---

## Frontend — API Service Layer

```javascript
// src/api/client.js
import axios from 'axios';

const client = axios.create({ baseURL: 'http://localhost:5001/api' });

client.interceptors.request.use(cfg => {
  const token = localStorage.getItem('gs_token');
  if (token) cfg.headers.Authorization = `Bearer ${token}`;
  return cfg;
});

client.interceptors.response.use(
  res => res,
  err => {
    if (err.response?.status === 401) {
      localStorage.removeItem('gs_token');
      window.location.href = '/';
    }
    return Promise.reject(err);
  }
);

export default client;
```

```javascript
// src/api/auth.js
import client from './client';
export const login  = (username, password) => client.post('/auth/login', { username, password });
export const logout = () => client.post('/auth/logout');

// src/api/scada.js  ← NEW
import client from './client';
import * as signalR from '@microsoft/signalr';

export const getTelemetry  = ()     => client.get('/scada/telemetry');
export const getSolarNodes = ()     => client.get('/scada/solar-nodes');
export const recordReading = (data) => client.post('/scada/telemetry', data);
export const updateSolar   = (id, data) => client.patch(`/scada/solar-nodes/${id}`, data);

export const connectScadaLive = (onUpdate) => {
  const conn = new signalR.HubConnectionBuilder()
    .withUrl('http://localhost:5001/hubs/grid',
      { accessTokenFactory: () => localStorage.getItem('gs_token') })
    .withAutomaticReconnect()
    .build();

  conn.start().then(() => conn.invoke('JoinScada'));
  conn.on('scadaUpdate', onUpdate);

  return () => conn.stop();  // cleanup
};

// src/api/incidents.js
import client from './client';
export const getIncidents   = (status)     => client.get('/incidents', { params: { status } });
export const getIncident    = (id)         => client.get(`/incidents/${id}`);
export const createIncident = (data)       => client.post('/incidents', data);
export const updateIncident = (id, data)   => client.put(`/incidents/${id}`, data);
export const updateStatus   = (id, status) => client.patch(`/incidents/${id}/status`, { status });
export const deleteIncident = (id)         => client.delete(`/incidents/${id}`);

// src/api/users.js
import client from './client';
export const getUsers     = ()        => client.get('/users');
export const createUser   = (data)    => client.post('/users', data);
export const updateUser   = (id, d)   => client.put(`/users/${id}`, d);
export const toggleActive = (id)      => client.patch(`/users/${id}/toggle-active`);
export const deleteUser   = (id)      => client.delete(`/users/${id}`);

// src/api/dashboard.js
import client from './client';
import * as signalR from '@microsoft/signalr';

export const getStats = () => client.get('/dashboard/stats');

export const connectDashboardLive = (onUpdate) => {
  const conn = new signalR.HubConnectionBuilder()
    .withUrl('http://localhost:5001/hubs/grid',
      { accessTokenFactory: () => localStorage.getItem('gs_token') })
    .withAutomaticReconnect()
    .build();

  conn.start().then(() => conn.invoke('JoinDashboard'));
  conn.on('dashboardUpdate', onUpdate);

  return () => conn.stop();
};
```

---

## Advanced C# Features Used

| Feature | Where |
|---------|-------|
| Async/Await | All repository & service methods |
| CQRS with MediatR | All Incident & User operations |
| Generic Repository `IRepository<T>` | Base interface for all repos |
| Dependency Injection | All services registered in `Program.cs` |
| JWT + Role Claims | `AuthService`, policy-based controllers |
| Middleware Pipeline | `ExceptionMiddleware`, auth, CORS |
| SignalR | `GridHub` — dual groups (dashboard + scada) |
| Background Services | `TelemetryBroadcaster` — sine-wave RTU simulation |
| EF Core + Npgsql | `GridShieldDbContext`, migrations |
| FluentValidation | All request DTOs |
| AutoMapper | Entity ↔ DTO conversions |
| Serilog | Structured logging |
| Record Types | MediatR commands & queries |
| Pattern Matching | Status update switch expressions |
| LINQ | Query logic + telemetry aggregation in broadcaster |
| BCrypt | Password hashing in `AuthService` |

---

## This-Week Schedule (April 14–18, 2026)

### Day 1 — Monday, April 14
- [ ] Create solution + 4 projects (`API`, `Core`, `Infrastructure`, `Tests`)
- [ ] Install all NuGet packages
- [ ] Define all entities (`User`, `Incident`, `RtuTelemetry`, `SolarNode`), enums, DTOs, interfaces in `Core`
- [ ] Set up `GridShieldDbContext` with Supabase connection string and seed data
- [ ] Run initial migration → verify all 4 tables appear in Supabase dashboard

### Day 2 — Tuesday, April 15
- [ ] Implement `AuthService` (BCrypt + JWT generation)
- [ ] Implement `UserRepository`
- [ ] Build `AuthController` (login + logout) and `UsersController` (full CRUD + toggle-active)
- [ ] Register JWT middleware and authorization policies in `Program.cs`
- [ ] Test login via Swagger → confirm token and role claims returned

### Day 3 — Wednesday, April 16
- [ ] Implement CQRS commands/queries for Incidents (MediatR handlers)
- [ ] Implement `IncidentRepository`
- [ ] Build `IncidentsController` (full CRUD + status PATCH)
- [ ] Add `ExceptionMiddleware` and Serilog structured logging

### Day 4 — Thursday, April 17
- [ ] Implement `ScadaRepository` (latest readings per RTU + solar nodes)
- [ ] Build `ScadaController` (GET telemetry, GET solar-nodes, POST reading, PATCH solar node)
- [ ] Set up `GridHub` with `JoinDashboard` and `JoinScada` groups
- [ ] Implement `TelemetryBroadcaster` BackgroundService (sine-wave RTU + solar push every 2s)
- [ ] Build `DashboardController` (aggregated stats endpoint)
- [ ] Write unit tests in `GridShield.Tests` (minimum 3: CreateIncident, Login success, Login failure)

### Day 5 — Friday, April 18
- [ ] Trim React `App.jsx` to 5 modules (remove 9 unused modules)
- [ ] Add `src/api/` service layer (all 5 service files + client.js)
- [ ] Install `@microsoft/signalr` in frontend
- [ ] Wire Login page to real `AuthController` (remove hardcoded USERS array)
- [ ] Wire Dashboard to `/api/dashboard/stats` + SignalR `dashboardUpdate`
- [ ] Wire SCADA Monitor to `/api/scada/telemetry` + `/api/scada/solar-nodes` + SignalR `scadaUpdate`
- [ ] Wire Incident Management to full CRUD API
- [ ] Wire User Management to admin API
- [ ] Final end-to-end test with all 4 roles
- [ ] Verify Supabase tables have live data

---

## Credentials (unchanged from original app)

| Username | Password | Role |
|----------|----------|------|
| admin | admin123 | Administrator |
| operator | ops2026 | Grid Operator |
| analyst | analyst26 | Security Analyst |
| engineer | eng2026 | Field Engineer |

# SwiftRoute — Deep Explanation of Key Concepts
### Backend + Frontend with Your Actual Code Examples

---

# PART 1 — BACKEND KEY CONCEPTS

---

## 1. Dependency Injection (DI)

### What is it?
Dependency Injection means: instead of a class creating its own dependencies (using `new`), the framework
creates and "injects" them for you. The class just asks for what it needs in its constructor — it doesn't
care how they are made.

**Without DI (bad):**
```csharp
public class BookingService {
    private readonly AppDbContext _db = new AppDbContext(); // ❌ creates its own — tightly coupled
}
```

**With DI (good):**
```csharp
public class BookingService {
    private readonly AppDbContext _db;
    public BookingService(AppDbContext db) { _db = db; } // ✅ framework gives it — loosely coupled
}
```

### In Your Project — Program.cs

This is where all DI registrations happen. Every `AddScoped<>` line is saying:
"When anyone asks for THIS interface, give them THIS implementation."

```csharp
// Program.cs

builder.Services.AddScoped<IPasswordService, PasswordService>();
builder.Services.AddScoped<ITokenService,    TokenService>();
builder.Services.AddScoped<IUserService,     UserService>();
builder.Services.AddScoped<IBusService,      BusService>();
builder.Services.AddScoped<IBookingService,  BookingService>();
builder.Services.AddScoped<IAuditLogService, AuditLogService>();

// Single generic repository covers ALL 10 entities
builder.Services.AddScoped(typeof(IRepository<>), typeof(Repository<>));
```

### In Your Project — Controller receiving injected services

```csharp
// BookingsController.cs
public class BookingsController : ControllerBase
{
    private readonly IBookingService _bookings;       // ← interface, not concrete class
    private readonly AppDbContext    _db;

    // Framework reads this constructor and injects both automatically
    public BookingsController(IBookingService bookings, AppDbContext db)
    {
        _bookings = bookings;
        _db = db;
    }
}
```

### In Your Project — Service receiving injected services

```csharp
// BookingService.cs — has 5 injected dependencies
public class BookingService : IBookingService
{
    private readonly IRepository<Booking>          _bookings;
    private readonly IRepository<BookingPassenger> _passengers;
    private readonly IRepository<Payment>          _payments;
    private readonly IRepository<BusSchedule>      _schedules;
    private readonly AppDbContext                  _db;

    public BookingService(
        IRepository<Booking> bookings,
        IRepository<BookingPassenger> passengers,
        IRepository<Payment> payments,
        IRepository<BusSchedule> schedules,
        AppDbContext db)
    {
        _bookings   = bookings;
        _passengers = passengers;
        _payments   = payments;
        _schedules  = schedules;
        _db         = db;
    }
}
```

### Service Lifetimes (very important)

| Lifetime      | Created           | Destroyed          | Used For               |
|---------------|-------------------|--------------------|------------------------|
| `AddScoped`   | Once per request  | End of request     | All our services (DB)  |
| `AddSingleton`| Once ever         | App shutdown       | Config, caches         |
| `AddTransient`| Every time asked  | After each use     | Lightweight helpers    |

**Why Scoped for services?**
User A's request gets its own `BookingService` with its own `AppDbContext`.
User B's request gets a completely separate one. No data bleeds between users.

---

## 2. Generics

### What is it?
Generics let you write one piece of code that works for many different types.
Instead of writing a separate repository for Bus, Booking, User, etc., you write ONE class
with a placeholder `T` that gets replaced with the actual type at runtime.

### In Your Project — IRepository<TEntity>

```csharp
// Interfaces/IRepository.cs

// TEntity is a placeholder. "where TEntity : class" means T must be a class (not int/bool)
public interface IRepository<TEntity> where TEntity : class
{
    Task<TEntity?>              GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<IEnumerable<TEntity>>  GetAllAsync(CancellationToken ct = default);
    Task<IEnumerable<TEntity>>  FindAsync(Expression<Func<TEntity, bool>> predicate, CancellationToken ct = default);
    Task<TEntity>               AddAsync(TEntity entity, CancellationToken ct = default);
    Task                        UpdateAsync(TEntity entity, CancellationToken ct = default);
    Task                        RemoveAsync(TEntity entity, CancellationToken ct = default);
}
```

### In Your Project — Repository<TEntity> Implementation

```csharp
// Repositories/Repository.cs
public class Repository<TEntity> : IRepository<TEntity> where TEntity : class
{
    protected readonly AppDbContext  _db;
    protected readonly DbSet<TEntity> _set;

    public Repository(AppDbContext db)
    {
        _db  = db;
        _set = _db.Set<TEntity>();  // ← EF Core gives us the correct table for T
    }

    public async Task<TEntity?> GetByIdAsync(Guid id, CancellationToken ct = default)
        => await _set.FindAsync(new object?[] { id }, ct);  // works for Bus, Booking, User — any T

    public async Task<IEnumerable<TEntity>> FindAsync(
        Expression<Func<TEntity, bool>> predicate, CancellationToken ct = default)
        => await _set.AsNoTracking().Where(predicate).ToListAsync(ct);
}
```

### How it's used in Services

```csharp
// BusService.cs — uses 3 different generic repositories
public class BusService : IBusService
{
    private readonly IRepository<Bus>         _buses;      // IRepository<T> where T = Bus
    private readonly IRepository<BusOperator> _operators;  // IRepository<T> where T = BusOperator
    private readonly IRepository<User>        _users;      // IRepository<T> where T = User

    // Each one works exactly the same way — same methods, different type
    var buses     = await _buses.GetAllAsync(ct);         // returns IEnumerable<Bus>
    var operators = await _operators.FindAsync(o => o.UserId == userId, ct); // returns IEnumerable<BusOperator>
}
```

### DI Registration for Generic

```csharp
// Program.cs — ONE line registers it for ALL entity types
builder.Services.AddScoped(typeof(IRepository<>), typeof(Repository<>));

// Without generics, you'd need 10 lines like:
// builder.Services.AddScoped<IRepository<Bus>,         Repository<Bus>>();
// builder.Services.AddScoped<IRepository<Booking>,     Repository<Booking>>();
// ... 8 more
```

---

## 3. Middleware Pipeline

### What is it?
Middleware is code that runs on EVERY request before/after it reaches your controller.
Think of it as a series of checkpoints a request passes through. Each middleware can:
- Do something before passing to the next (e.g., validate token)
- Do something after the next returns (e.g., log the response)
- Short-circuit and return early (e.g., 401 if no token)

### Pipeline Order in Your Project

```csharp
// Program.cs — ORDER MATTERS

app.UseHttpsRedirection();         // 1. Force HTTPS
app.UseCors();                     // 2. Allow cross-origin (Angular → API)
app.UseMiddleware<AuditMiddleware>();            // 3. Start timer, log writes
app.UseMiddleware<GlobalExceptionMiddleware>();  // 4. Catch all errors
app.UseAuthentication();           // 5. Read JWT, populate User claims
app.UseAuthorization();            // 6. Check [Authorize] attributes
app.MapControllers();              // 7. Route to correct controller action
```

**Why does order matter?** If you put `UseAuthentication` BEFORE `UseCors`, then browser
preflight OPTIONS requests fail because they have no token. Always CORS first.

### In Your Project — AuditMiddleware

```csharp
// Middlewares/AuditMiddleware.cs
public class AuditMiddleware
{
    private readonly RequestDelegate _next;   // ← the NEXT middleware in the chain

    public AuditMiddleware(RequestDelegate next, ILogger<AuditMiddleware> logger)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext ctx)
    {
        var sw = Stopwatch.StartNew();     // start timer
        try
        {
            await _next(ctx);              // ← run all the other middlewares + controller
            sw.Stop();

            // AFTER the controller runs — log the result
            if (ctx.Request.Path.StartsWithSegments("/api") &&
                ctx.Request.Method is "POST" or "PUT" or "PATCH" or "DELETE")
            {
                await WriteAuditAsync(ctx, sw.ElapsedMilliseconds);
            }
        }
        catch (Exception ex)
        {
            // If anything in the chain throws, catch it here and log as Error
            await WriteErrorAsync(ctx, ex, sw.ElapsedMilliseconds);
            throw;  // ← re-throw so GlobalExceptionMiddleware also handles it
        }
    }
}
```

The key pattern: `await _next(ctx)` is where the entire rest of the pipeline runs.
Code before it = runs before the controller. Code after it = runs after.

---

## 4. Interfaces and Abstraction

### What is it?
An interface defines a CONTRACT — what methods a class must have, without specifying HOW.
You program to the interface, not the implementation. This is the "I" in SOLID.

### In Your Project

```csharp
// Interfaces/IBookingService.cs — the CONTRACT
public interface IBookingService
{
    Task<BookingResponseDto> CreateAsync(Guid userId, CreateBookingRequestDto dto, CancellationToken ct);
    Task<IEnumerable<BookingResponseDto>> GetMyAsync(Guid userId, CancellationToken ct);
    Task<bool> CancelAsync(Guid userId, Guid bookingId, bool allowPrivileged, CancellationToken ct);
    Task<BookingResponseDto?> PayAsync(Guid userId, Guid bookingId, decimal amount, string providerRef, bool allowPrivileged, CancellationToken ct);
}

// Services/BookingService.cs — the IMPLEMENTATION
public class BookingService : IBookingService
{
    public async Task<BookingResponseDto> CreateAsync(...) { /* actual logic here */ }
}
```

**Why is this powerful?**
- In tests, you mock the interface: `Mock<IBookingService>()` — no database needed
- You can swap implementations without changing the controller
- Controller only knows `IBookingService` exists — doesn't care it's `BookingService`

---

## 5. async/await and CancellationToken

### What is it?
`async/await` lets your code not block threads while waiting for database or network calls.
`CancellationToken` lets the client cancel the request mid-way (e.g., user closes the browser).

### In Your Project

```csharp
// BookingService.cs
public async Task<BookingResponseDto> CreateAsync(
    Guid userId, CreateBookingRequestDto dto, CancellationToken ct = default)
{
    // All these are non-blocking — thread is freed while waiting for DB
    var schedule = await _db.BusSchedules
        .Include(s => s.Bus)
        .AsNoTracking()
        .FirstOrDefaultAsync(s => s.Id == dto.ScheduleId, ct);  // ← ct passed through

    // If browser tab is closed, ct is cancelled → this throws and stops work
    await using var tx = await _db.Database.BeginTransactionAsync(IsolationLevel.Serializable, ct);
    await _db.SaveChangesAsync(ct);
    await tx.CommitAsync(ct);
}
```

**Every database call receives `ct`** — so if a customer's browser closes mid-booking,
the entire transaction is cancelled cleanly. No partial bookings in the database.

---

## 6. Entity Framework Core — Code-First, Relationships, Concurrency

### Code-First
You define models in C#. EF Core generates the SQL. Run `dotnet ef migrations add` and
`dotnet ef database update` — tables are created automatically.

### Relationships in Your Project

```csharp
// AppDbContext.cs — relationships defined in OnModelCreating

// ONE-TO-MANY: Operator has many Buses. Delete operator → delete all their buses.
modelBuilder.Entity<Bus>(e => {
    e.HasOne(b => b.Operator)
     .WithMany(o => o.Buses)
     .HasForeignKey(b => b.OperatorId)
     .OnDelete(DeleteBehavior.Cascade);   // ← cascade delete
});

// ONE-TO-ONE: Booking has one Payment
modelBuilder.Entity<Booking>(e => {
    e.HasOne(b => b.Payment)
     .WithOne(p => p.Booking)
     .HasForeignKey<Payment>(p => p.BookingId)
     .OnDelete(DeleteBehavior.Cascade);
});

// RESTRICT: Can't delete a Stop if a RouteStop references it
modelBuilder.Entity<RouteStop>(e => {
    e.HasOne(rs => rs.Stop)
     .WithMany(s => s.RouteStops)
     .HasForeignKey(rs => rs.StopId)
     .OnDelete(DeleteBehavior.Restrict);  // ← prevents accidental data loss
});
```

### Concurrency Token (RowVersion) in Your Project

```csharp
// BaseEntity.cs — every entity has this
public byte[]? RowVersion { get; set; }

// AppDbContext.cs — configured as concurrency token
foreach (var entity in modelBuilder.Model.GetEntityTypes()) {
    var rowVersion = entity.FindProperty(nameof(BaseEntity.RowVersion));
    if (rowVersion != null) {
        rowVersion.IsConcurrencyToken = true;           // ← EF uses this for conflict detection
        rowVersion.ValueGenerated = ValueGenerated.OnAddOrUpdate;  // DB auto-updates it
    }
}
```

**How it prevents conflicts:**
1. User A reads Booking — RowVersion = `0x001`
2. User B reads same Booking — RowVersion = `0x001`
3. User A saves — DB updates RowVersion to `0x002`
4. User B tries to save with old `0x001` → EF Core throws `DbUpdateConcurrencyException`
   → second update is rejected — no silent data loss

---

## 7. JWT Authentication — How It Actually Works

### Token structure
A JWT has 3 parts separated by dots: `header.payload.signature`
All 3 are base64-encoded. The signature is what makes it tamper-proof.

### In Your Project — TokenService.cs

```csharp
public (string token, DateTime expiresAtUtc) GenerateAccessToken(User user)
{
    string key = _config["Jwt:Key"];          // Secret key from appsettings
    var expires = DateTime.UtcNow.AddMinutes(60);

    // What goes INSIDE the token
    var claims = new List<Claim>
    {
        new(JwtRegisteredClaimNames.Sub,        user.Id.ToString()),   // ← user's GUID
        new(JwtRegisteredClaimNames.UniqueName, user.Username),
        new(JwtRegisteredClaimNames.Email,      user.Email),
        new(ClaimTypes.Name,                    user.FullName),
        new(ClaimTypes.Role,                    user.Role),            // ← "Admin", "Customer" etc
        new(JwtRegisteredClaimNames.Jti,        Guid.NewGuid().ToString()) // ← unique token ID
    };

    var signingKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(key));
    var creds = new SigningCredentials(signingKey, SecurityAlgorithms.HmacSha256);

    // Build and sign the token
    var token = new JwtSecurityToken(
        claims: claims,
        expires: expires,
        signingCredentials: creds);

    return (new JwtSecurityTokenHandler().WriteToken(token), expires);
}
```

### Validation in Program.cs

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options => {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,          // must be signed with our key
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(jwtKey)),
            ValidateLifetime = true,                  // must not be expired
            ClockSkew = TimeSpan.FromMinutes(1)       // 1 min tolerance for clock differences
        };
    });
```

### Reading claims in controllers

```csharp
// BookingsController.cs
private Guid GetUserId()
{
    var sub = User.FindFirstValue(JwtRegisteredClaimNames.Sub)
           ?? User.FindFirstValue(ClaimTypes.NameIdentifier);
    return Guid.TryParse(sub, out var id) ? id : Guid.Empty;
}
// "User" here is HttpContext.User — automatically populated by UseAuthentication()
// from the JWT in the request header
```

---

## 8. LINQ and Expression Predicates

### What is it?
LINQ (Language Integrated Query) lets you write SQL-like queries in C#.
`Expression<Func<T, bool>>` is a lambda that EF Core translates to a SQL WHERE clause.

### In Your Project

```csharp
// Repository.cs — takes an expression predicate
public async Task<IEnumerable<TEntity>> FindAsync(
    Expression<Func<TEntity, bool>> predicate, CancellationToken ct = default)
    => await _set.AsNoTracking().Where(predicate).ToListAsync(ct);

// Usage in BusService.cs — EF Core translates this to:
// SELECT * FROM Buses WHERE OperatorId = 'some-guid'
var list = await _buses.FindAsync(b => b.OperatorId == op.Id, ct);

// Usage in BookingService.cs
// SELECT * FROM BookingPassengers
// JOIN Bookings ON ...
// WHERE SeatNo IN ('1A','2B') AND ScheduleId = '...' AND Status NOT IN (3,4)
var takenSeats = await _db.BookingPassengers
    .Where(bp => requestedSeats.Contains(bp.SeatNo))
    .Join(_db.Bookings, bp => bp.BookingId, b => b.Id,
          (bp, b) => new { bp.SeatNo, b.Status, b.ScheduleId })
    .Where(x => x.ScheduleId == dto.ScheduleId
             && x.Status != BookingStatus.Cancelled
             && x.Status != BookingStatus.Refunded)
    .Select(x => x.SeatNo)
    .ToListAsync(ct);
```

---

# PART 2 — FRONTEND KEY CONCEPTS

---

## 9. Angular Signals — Reactive State

### What is it?
A Signal is a box that holds a value. When the value changes, Angular automatically
updates every part of the UI that reads it. No manual subscriptions, no `markForCheck()`.

### Three types in your project

```typescript
// 1. Writable signal — you control its value
loading = signal(true);          // initial value
loading.set(false);              // replace value
loading.update(v => !v);        // transform value

// 2. Computed signal — derived automatically from another signal
// Re-calculates ONLY when loading() or bookings() changes
filtered = computed(() => {
    const f = this.activeFilter();
    const all = this.bookings();
    return f === 'all' ? all : all.filter(b => ...);
});

// 3. Read-only signal — expose internally-writable signal as read-only
private _currentUser = signal<CurrentUser | null>(null);
currentUser = this._currentUser.asReadonly(); // outsiders can READ but not SET

// All computed signals — updated automatically when _currentUser changes
isLoggedIn = computed(() => !!this._currentUser());
role       = computed(() => this._currentUser()?.role ?? null);
isAdmin    = computed(() => this.role() === 'Admin');
```

**In templates — just call the signal like a function:**
```html
@if (loading()) { <spinner/> }
@for (b of filtered(); track b.id) { <booking-card [booking]="b"/> }
<p>Total: ₹{{ total() }}</p>
```

---

## 10. Standalone Components and Lazy Loading

### Standalone Components
No NgModule needed. Each component declares its own imports.

```typescript
// seat-selection.component.ts
@Component({
    selector: 'app-seat-selection',
    standalone: true,                          // ← standalone
    imports: [CommonModule],                   // ← only what THIS component needs
    template: `...`
})
export class SeatSelectionComponent {}
```

### Lazy Loading in app.routes.ts

```typescript
// app.routes.ts
{
    path: 'admin',
    canActivate: [authGuard, roleGuard('Admin')],
    children: [
        {
            path: '',
            // ↓ This component's JavaScript is NOT downloaded until user navigates to /admin
            loadComponent: () =>
                import('./pages/admin/admin-dashboard/admin-dashboard.component')
                .then(m => m.AdminDashboardComponent)
        },
        {
            path: 'manage-users',
            loadComponent: () =>
                import('./pages/admin/manage-users/manage-users.component')
                .then(m => m.ManageUsersComponent)
        }
    ]
}
```

**Why lazy loading?** On first load, the browser only downloads the code needed for the
current page. The admin dashboard code (tens of kilobytes) is only downloaded when an
Admin actually navigates there. Makes initial page load much faster for customers.

---

## 11. HTTP Interceptor

### What is it?
An interceptor sits between your Angular services and the actual HTTP calls.
It can modify EVERY request going out and EVERY response coming in — in one place.

### In Your Project — authInterceptor

```typescript
// core/auth.interceptor.ts
export const authInterceptor: HttpInterceptorFn = (req, next) => {
    const auth   = inject(AuthService);
    const router = inject(Router);

    const token = auth.getToken();

    // Add Authorization header to EVERY outgoing request
    const authReq = token
        ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } })
        : req;  // ← clone because HttpRequest objects are immutable

    return next(authReq).pipe(
        catchError((err: HttpErrorResponse) => {
            // Handle 401 (token expired/invalid) for EVERY response in one place
            if (err.status === 401) {
                auth.logout();
                router.navigate(['/auth/login']);
            }
            return throwError(() => err);
        })
    );
};
```

**Without interceptor** — every service would need:
```typescript
// ❌ Duplicated in every service method
this.http.get('/api/buses', {
    headers: { Authorization: `Bearer ${this.auth.getToken()}` }
})
```

**With interceptor** — services are clean:
```typescript
// ✅ Just call the API — interceptor handles the header automatically
getAll(): Observable<BusResponse[]> {
    return this.http.get<BusResponse[]>(`${this.base}`);
}
```

### Registered in app.config.ts

```typescript
export const appConfig: ApplicationConfig = {
    providers: [
        provideHttpClient(withInterceptors([authInterceptor]))
        //                ↑ registers our interceptor globally
    ]
};
```

---

## 12. Route Guards

### What is it?
Guards are functions that decide whether Angular should allow navigation to a route.
They return `true` (allow) or a `UrlTree` (redirect elsewhere).

### Three guards in your project

```typescript
// core/guards.ts

// ── 1. authGuard — must be logged in ──────────────────────────
export const authGuard: CanActivateFn = () => {
    const auth = inject(AuthService);
    if (auth.isLoggedIn() && !auth.isTokenExpired()) return true;
    return inject(Router).createUrlTree(['/auth/login']); // ← UrlTree = redirect
};

// ── 2. noAuthGuard — must NOT be logged in ────────────────────
export const noAuthGuard: CanActivateFn = () => {
    const auth = inject(AuthService);
    if (!auth.isLoggedIn() || auth.isTokenExpired()) return true;

    // Already logged in — redirect to their homepage
    const role = auth.role();
    if (role === 'Admin')           return inject(Router).createUrlTree(['/admin']);
    if (role === 'Operator')        return inject(Router).createUrlTree(['/operator']);
    if (role === 'PendingOperator') return inject(Router).createUrlTree(['/home']);
    return inject(Router).createUrlTree(['/home']);
};

// ── 3. roleGuard — must have specific role ────────────────────
export function roleGuard(...allowedRoles: string[]): CanActivateFn {
    return () => {
        const role = inject(AuthService).role();
        if (role && allowedRoles.includes(role)) return true;
        if (!inject(AuthService).isLoggedIn())
            return inject(Router).createUrlTree(['/auth/login']);
        return inject(Router).createUrlTree(['/home']);
    };
}
```

### Applied in routes

```typescript
// app.routes.ts
{
    path: 'admin',
    canActivate: [authGuard, roleGuard('Admin')],  // ← both must pass
    children: [...]
}
{
    path: 'auth',
    children: [
        { path: 'login',    canActivate: [noAuthGuard], loadComponent: ... },
        { path: 'register', canActivate: [noAuthGuard], loadComponent: ... },
    ]
}
```

**Both guards run in order**: `authGuard` checks if logged in first.
If that passes, `roleGuard('Admin')` checks if they're actually Admin.
A Customer can't reach `/admin` even if they're logged in.

---

## 13. Reactive Forms and FormArray

### FormGroup — for simple forms

```typescript
// login.component.ts
form = this.fb.group({
    username: ['', [Validators.required]],
    password: ['', [Validators.required, Validators.minLength(6)]],
});

// Check if field is invalid AND has been touched (user interacted with it)
isInvalid(f: string): boolean {
    const c = this.form.get(f);
    return !!(c?.invalid && c?.touched);
}
```

### FormArray — for dynamic lists (your passenger form)

```typescript
// passenger-form.component.ts
form = this.fb.group({ passengers: this.fb.array([]) });

get passengersArray(): FormArray {
    return this.form.get('passengers') as FormArray;
}

ngOnInit() {
    // Dynamically add one FormGroup per selected seat
    d.selectedSeats.forEach(seat =>
        this.passengersArray.push(this.fb.group({
            name:   ['', [Validators.required, Validators.maxLength(150)]],
            age:    [null, [Validators.min(0), Validators.max(120)]],
            seatNo: [seat],   // ← pre-filled from seat selection
        }))
    );
}
```

```html
<!-- Template — dynamic rows per passenger -->
<div formArrayName="passengers">
    @for (pg of passengersArray.controls; track $index) {
        <div [formGroupName]="$index">
            <input formControlName="name" />
            <input formControlName="age" type="number" />
            <input formControlName="seatNo" readonly />
        </div>
    }
</div>
```

---

## 14. BookingStateService — Cross-Page State

### The Problem
Booking spans 3 pages (Seats → Passengers → Payment). Angular destroys each component
when you navigate away. How do you pass data across pages without URL params?

### The Solution — Injectable Singleton Signal

```typescript
// services/booking-state.service.ts
@Injectable({ providedIn: 'root' })  // ← 'root' = ONE instance for the whole app
export class BookingStateService {

    private _draft = signal<BookingDraft | null>(null);
    readonly draft = this._draft.asReadonly();  // ← expose as read-only

    setSchedule(schedule: ScheduleResponse): void {
        this._draft.set({ schedule, selectedSeats: [], passengers: [] });
    }

    setSeats(seats: string[]): void {
        // Keep existing schedule, replace seats
        this._draft.update(d => d ? { ...d, selectedSeats: seats } : null);
    }

    setPassengers(passengers: BookingPassengerDto[]): void {
        this._draft.update(d => d ? { ...d, passengers } : null);
    }

    clear(): void { this._draft.set(null); }
}
```

**Flow across 3 pages:**
```
SeatSelectionComponent:
    bookingState.setSchedule(schedule)     // sets the schedule
    bookingState.setSeats(['1A', '2B'])    // adds selected seats

PassengerFormComponent:
    const draft = this.bookingState.draft()  // reads schedule + seats
    bookingState.setPassengers([...])       // adds passenger names

BookingConfirmComponent:
    // Just uses booking.id from navigation — state is cleared after booking created
```

---

# PART 3 — HOW BACKEND AND FRONTEND ARE CONNECTED

---

## 15. The Complete Request Journey

Here is exactly what happens when you click "Search buses" on the home page:

```
Step 1: User clicks Search in HomeComponent
        form.value = { fromCity: 'Chennai', toCity: 'Bangalore', date: '2025-06-01' }
        router.navigate(['/search'], { queryParams: { from, to, date } })

Step 2: SearchResultsComponent calls the service
        this.scheduleSvc.searchByKeys({ fromCity, toCity, date })

Step 3: ScheduleService makes HTTP call
        this.http.post<PagedResult>('/api/schedules/search-by-keys', body)
        ← apiUrl is '/api' (from environment.ts)

Step 4: authInterceptor intercepts the request
        Checks: does this user have a token?
        Token exists → clones request, adds: Authorization: Bearer eyJhbGci...
        Request goes out to: /api/schedules/search-by-keys

Step 5: Angular dev server sees the request to /api/*
        proxy.config.json says: forward /api → http://localhost:5299
        Request becomes: POST http://localhost:5299/api/schedules/search-by-keys

Step 6: ASP.NET Core receives the request
        Middleware pipeline runs:
          → CORS: origin allowed ✓
          → AuditMiddleware: start timer
          → UseAuthentication: reads JWT, populates HttpContext.User
          → UseAuthorization: [AllowAnonymous] on this endpoint, passes through
          → Routes to SchedulesController.SearchByKeys()

Step 7: Controller calls service
        await _schedules.SearchByKeysAsync(dto, ct)

Step 8: Service queries SQL Server via EF Core
        var results = await _db.BusSchedules
            .Include(s => s.Bus)
            .Include(s => s.Route)
            .Where(s => cities match AND date matches)
            .ToListAsync(ct);

Step 9: Response travels back
        JSON → ASP.NET → proxy → Angular → Observable → Component → Signal → UI updates
```

---

## 16. Why apiUrl = '/api' and NOT 'http://localhost:5299' — Full Explanation

This is the most misunderstood part of the setup. Let me explain it completely.

### The Problem: CORS

When a browser makes a request, it checks if the request is "same-origin".
Origin = protocol + domain + port.

```
Angular runs on:   http://localhost:4200
Backend runs on:   http://localhost:5299
                   ↑ DIFFERENT PORT = DIFFERENT ORIGIN
```

If Angular calls `http://localhost:5299/api/buses` directly, the browser blocks it
with: **"Cross-Origin Request Blocked"**. This is a browser security policy.

### Option 1 (what we use): Angular Dev Proxy

```typescript
// environment.ts
export const environment = {
    production: false,
    apiUrl: '/api',   // ← just a path, not a full URL
};
```

```json
// proxy.config.json
{
    "/api": {
        "target": "http://localhost:5299",
        "secure": false,
        "changeOrigin": true
    }
}
```

**What happens:**
```
Browser → http://localhost:4200/api/buses    ← SAME ORIGIN (port 4200)
              ↓
Angular dev server sees /api → proxies to http://localhost:5299/api/buses
              ↓
Backend receives the request (from the server-side proxy, not the browser)
              ↓
Response comes back through Angular → Browser
Browser never directly talks to port 5299 → No CORS issue
```

The browser thinks it's talking to the Angular server on port 4200 the whole time.
The proxy is invisible to the browser.

### Option 2 (alternative): Hardcode the URL

```typescript
// ❌ This also works but is worse:
export const environment = {
    production: false,
    apiUrl: 'http://localhost:5299/api',
};
```

This would require CORS to be configured on the backend to allow `http://localhost:4200`.
Our backend does have `AllowAnyOrigin()` so it technically works — but you'd need to
remember to change this URL every time you deploy. With the proxy approach, you only
ever change `proxy.config.json`, not the app code.

### Production environment

```typescript
// environment.prod.ts (when you deploy)
export const environment = {
    production: true,
    apiUrl: 'https://your-real-api.com/api',  // ← real URL in production
};
```

In production there IS no dev proxy — so you need the real URL.
Angular automatically uses `environment.prod.ts` when you run `ng build --configuration production`.

### Why it works right now with the comment

Your `environment.ts` has:
```typescript
// Use the dev proxy: Angular -> /api -> http://localhost:5299
apiUrl: '/api',
```

Every service uses `environment.apiUrl` as the base:
```typescript
// booking.service.ts
private base = `${environment.apiUrl}/bookings`;
// → '/api/bookings'
// → Angular dev server proxies this to http://localhost:5299/api/bookings
```

**Summary of why `/api` not `http://localhost:5299`:**

| Approach | CORS issue? | Works in prod? | Code to change for deployment |
|----------|-------------|----------------|-------------------------------|
| `/api` + proxy.config.json | ❌ None | ✅ Yes (use env.prod.ts) | Only environment.prod.ts |
| `http://localhost:5299/api` | ⚠️ Needs CORS | ❌ No | Every environment file |

---

## 17. The Environment File Strategy

```
src/
  environments/
    environment.ts        ← used by ng serve (development)
    environment.prod.ts   ← used by ng build (production)
```

**`angular.json` tells Angular which to use:**
```json
"configurations": {
    "production": {
        "fileReplacements": [{
            "replace": "src/environments/environment.ts",
            "with": "src/environments/environment.prod.ts"
        }]
    }
}
```

When you run `ng build --configuration production`, Angular swaps `environment.ts` with
`environment.prod.ts` automatically. Your code always imports from `environment.ts` — the
swap happens at build time. You never change service code for different environments.

---

## QUICK REFERENCE — Key Patterns

| Pattern | Backend | Frontend |
|---------|---------|----------|
| Dependency Injection | `builder.Services.AddScoped<IBookingService, BookingService>()` | `inject(AuthService)` in components |
| Interface/Contract | `IRepository<T>`, `IBookingService` | `Observable<T>` from HttpClient |
| Generics | `IRepository<TEntity>`, `Repository<TEntity>` | `signal<BookingDraft \| null>()`, `Observable<T>` |
| Cross-cutting concern | `AuditMiddleware`, `GlobalExceptionMiddleware` | `authInterceptor` |
| State | `AppDbContext` (DB) | `BookingStateService` (signal), `AuthService` (localStorage) |
| Route protection | `[Authorize(Roles="Admin")]` | `authGuard`, `roleGuard('Admin')` |
| Async | `async/await` + `CancellationToken` | `Observable` + `.subscribe()` |
| Type safety | C# strong typing, LINQ expressions | TypeScript interfaces, `UserRole` union type |

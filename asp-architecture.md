# Kiến trúc dự án ASP.NET Core (Api + Razor Pages Web)

Chuẩn rút từ một repo mẫu (khuôn chính, BẮT BUỘC theo) và vài repo cũ (chỉ lấy phần trùng khuôn);
đường dẫn repo mẫu và danh sách repo cũ ghi ở `~/.claude/local.md` (file local, không nằm trong git).

**Khi nào phải đọc và tuân theo file này:**
- Dựng project/solution ASP mới, hoặc thêm project, tầng, bảng/entity, repository, service, controller,
  client typed, trang Razor, tài nguyên i18n vào project đang có.
- **Project cũ** (tạo trước khi có file này, hoặc theo khuôn khác): mọi lần **sửa lớn (refactor)** hay
  **tạo mới bất kỳ thứ gì** cũng phải đọc file này và làm theo kiến trúc ở đây. Nếu phần định làm **xung
  đột với kiến trúc đang có** của project cũ (vd repo chỉ có một project `DataBase`, service đang cầm
  thẳng `DbContext`, repository nhận/trả Dbo, bảng đang snake_case…) thì **HỎI người dùng** chọn: làm theo chuẩn mới (kèm phạm vi
  phải chuyển đổi) hay giữ khuôn cũ cho lần này — KHÔNG tự quyết, cũng KHÔNG âm thầm trộn hai khuôn.
- Phân vân "code này để ở đâu / đặt tên gì".

Ba file đi kèm, KHÔNG lặp lại nội dung của nhau: bẫy Razor/AJAX ở `~/.claude/asp.md`, bẫy EF/migration ở
`~/.claude/database-EF.md`, quy ước C#/namespace/OOP ở `~/.claude/csharp.md`.

---

## 1. Bố cục repo và solution

```
<repo>/
├── README.md, docs/, samples/, script tiện ích
└── src/
    ├── <Prefix>.slnx                 # solution mới (XML), KHÔNG dùng .sln
    ├── <Prefix>.slnLaunch            # profile F5 nhiều project (VS)
    ├── Directory.Build.props         # PathMap ẩn đường dẫn máy build (mọi project, mọi config)
    ├── Directory.Build.targets       # siết Release: DebugType/EnableSourceLink=false/... (điều kiện $(Configuration) BẮT BUỘC ở .targets)
    ├── Directory.Packages.props      # Central Package Management: MỌI version NuGet khai ở đây, csproj KHÔNG ghi Version
    ├── Directory.Build.rsp           # -maxCpuCount:2 -nodeReuse:false — CỤC BỘ máy, .gitignore, KHÔNG commit
    ├── ProjectBuildProperties.targets# LangVersion/Nullable/ImplicitUsings/PathMap — mỗi csproj <Import> TƯỜNG MINH, KHÔNG đặt TargetFramework ở đây
    ├── shared/                       # dùng chung nhiều ứng dụng trong repo (Contracts, *.Core)
    ├── <app>/                        # mỗi ứng dụng một thư mục (vd central/, node/)
    │   └── tests/
    └── tools/
```

- Solution folder đánh số để giữ thứ tự hiển thị: `1.shared`, `2.<app>` → `2.1.Shared`, `2.2.Database`,
  `2.3.Services` (gồm cả `<P>.BackgroundServices`), `2.4.Host`, `2.9.tests`, `4.tools`. File dùng chung
  nằm ở folder `0.Solution Items`.
- Tên project = `<Prefix>.<Area>.<Layer>[.<Feature>]`, vd `MyApp.Node.Services.Product`,
  `MyApp.Platform.Database.PostgreSql`. `RootNamespace` của nhóm feature TRÙNG namespace tầng cha
  (`MyApp.Node.Services`) để chẻ project không đổi namespace.
- Mỗi csproj **tự khai `TargetFramework`**; chỉ project cần API Windows mới `net10.0-windows`, và không kéo
  cả cây đa nền tảng theo (tách phần Windows ra project riêng, host tham chiếu thẳng, không qua facade).
- `Directory.Packages.props` là chỗ ghi **lý do** của mỗi ghim version bằng comment — ai nâng gói sau
  đọc là biết bẫy.

---

## 2. Các project và luật phụ thuộc (BẮT BUỘC)

Cây tầng của MỘT ứng dụng (đọc từ dưới lên = từ không phụ thuộc gì tới host):

| Project | Chứa gì | Được tham chiếu | CẤM tham chiếu |
|---|---|---|---|
| `<P>.Contracts` (shared) | enum + attribute mã dây, hằng quyền/claim, `ApiError`/`ApiResponse<T>`/`PagedResult<T>`, `<P>JsonSettings`, prompt/resource nhúng | Newtonsoft | mọi thứ khác |
| `<P>.ViewModels` | VM request/response (`record`), thư mục theo feature | Contracts, Newtonsoft (chỉ để `[JsonProperty]`) | Entities, EF, Services |
| `<P>.Database.Entities` | POCO `*Dbo` thuần, `[MaxLength]` trên property | Contracts (chỉ enum/hằng) | **KHÔNG PackageReference nào**, không EF |
| `<P>.Database.<Provider>` (`PostgreSql`, `SqlServer`) | `EF/<Name>DbContext.cs`, `EF/<Name>DbContextFactory.cs`, `Configurations/*Configuration.cs`, `Migrations/`, `EF/<Provider>SchemaInitializer.cs`, `Extensions/<Name>DataSeeder.cs`, `efdesign.json` | Entities, EF + provider, Configuration.Json (design-time) | Services, Repositories |
| `<P>.Repositories.Abstractions` | `Interfaces/I*Repository.cs`, `Models/` (kiểu nội bộ không được lọt ra API, xem 4.3) | ViewModels, Contracts | **Entities** (Dbo không có mặt trong chữ ký), **KHÔNG EF**, không `IQueryable`/`DbSet` trong chữ ký |
| `<P>.Repositories` | `*Repository.cs` cài đặt EF, `Mappings/<X>Mapping.cs` (Dbo ↔ RecordVM), helper truy vấn dùng chung (`DocumentPurge`) | Repositories.Abstractions, Database.<Provider> (qua đó thấy Entities) | Services |
| `<P>.Services.Abstractions` | interface dùng XUYÊN feature + model/enum của chúng (`ITokenService`, `IChatTool`, `DocumentIngestRequest`) | ViewModels, Contracts | Repositories, Database, **Entities** |
| `<P>.Services.Common` | đáy cây service: luật nghiệp vụ dùng chung (`Rules/`), lỗi (`Common/Errors/`), helper, client Central | Services.Abstractions, Repositories.Abstractions, Integration.* | **KHÔNG tham chiếu ngược feature nào**, **Entities** |
| `<P>.Services.<Feature>` | một project mỗi nhóm nghiệp vụ (Auth, Product, Documents, Chat…) | Services.Common, Services.Abstractions, Repositories.Abstractions, ViewModels, Contracts | **`<P>.Repositories`, `<P>.Database.<Provider>`, `<P>.Database.Entities`** (DbContext và Dbo không tồn tại ở đây để dùng nhầm) |
| `<P>.Services` (facade) | `AppServiceCollectionExtensions.AddApplicationServices()` gom mọi feature | mọi Services.<Feature> | — |
| `<P>.BackgroundServices` | MỌI hosted service (`BackgroundService`/`IHostedService`) của tiến trình Api, kể cả tiện ích chỉ-Debug; file phẳng ở gốc project, namespace = tên project. **Không có** `Add…Services()` ở đây — host tự gọi `AddHostedService<>` (xem 4.5) | Services (facade — qua đó thấy luôn `Repositories.Abstractions`), `Microsoft.Extensions.Hosting.Abstractions` | `<P>.Repositories` (bản EF), Database.<Provider>, host Api |
| `<P>.Integration.Api` | client typed của chính API này: `Interfaces/I<App>Api.cs`, `Interfaces/I<Group>Controller.cs`, `Implements/<App>ApiImplement*.cs`, `Extensions/ServiceCollectionExtensions.cs` | ViewModels, Contracts, Integration.Core | Services, Entities |
| `<P>.Api` (host) | `Controllers/`, `Middleware/`, `Extensions/` (DI), `Infrastructure/`, `Program.cs` | Services, **BackgroundServices**, **Repositories** (bản EF, để đăng ký DI), Integration.Api (controller implement interface), Api.Core | — |
| `<P>.Web` (host Razor Pages) | `Pages/`, `Infrastructure/`, `wwwroot/` | **CHỈ** Integration.Api + ViewModels + Contracts (+ Resources) | **Services, Repositories, Database, Entities** — Web không đụng database |
| `<P>.Resources` | i18n: `Resources/<Group>Resource.cs` + `.vi.json`/`.en.json`, `ILocalizerService`, `JsonLocalizerWarmup`, `<P>Cultures` | Askmethat.Aspnet.JsonLocalizer | — |
| `<P>.Api.Core` / `<P>.Integration.Core` / `<P>.Auth.Core` (shared) | filter/model binder/`ApiErrorException`; `<P>ApiBase` + bóc vỏ `ApiResponse`; mail/TOTP/thiết bị | Contracts; `FrameworkReference Microsoft.AspNetCore.App` (KHÔNG Sdk.Web) | — |
| `tests/<P>.Testing` | fake dùng chung: `Fakes/InMemory<App>Db.cs`, `Fakes/Repositories/InMemory*Repository.cs`, `Fakes/Services/Fake*.cs` | Repositories.Abstractions, Services (facade), Entities (fake giữ hàng dạng Dbo bên trong và map ra RecordVM y như bản thật, chép nguồn `Mappings/` của Repositories vào) | **`<P>.Repositories`, EF** (fake THAY THẾ tầng đó) |
| `tests/<P>.Tests` | xunit, thư mục theo tầng (`Services/Db`, `Services/Ai`, `Database`, `Web`, `Tools/<Feature>`) | Testing, Services, BackgroundServices, Repositories, Web | — |

**BẮT BUỘC, không có ngoại lệ cho project mới:** tầng dữ liệu phải tách đủ bốn project
`Database.Entities` / `Database.<Provider>` / `Repositories.Abstractions` / `Repositories`. Khuôn cũ một
project `<P>.DataBase` (entity + DbContext + migration chung) và service cầm thẳng `DbContext`
(các repo cũ) **không được dùng nữa**. Lý do: service thấy `DbContext`
là sớm muộn có người viết LINQ trong service, test phải có database thật, và đổi provider phải sửa
cả cây; tách ra thì `Services.<Feature>` biên dịch được mà không có EF trong tầm tay, và test chạy
bằng repository giả trong bộ nhớ. Gặp project cũ theo khuôn cũ: xem mục "Khi nào phải đọc" ở đầu file —
hỏi người dùng trước khi refactor.

Hai cái tên nhìn giống nhau mà nghĩa khác nhau, đừng lẫn:
- `Services.Abstractions` = interface mà **nhiều feature** hoặc host cùng gọi. Interface chỉ feature
  đó dùng thì nằm trong `Services.<Feature>/<Feature>/Interfaces/`.
- `Repositories.Abstractions` = **mọi** interface repository, không có ngoại lệ (feature nào cũng cần).

---

## 3. Luồng một request (ví dụ: tạo sản phẩm)

1. Trình duyệt `fetch POST /product?handler=Create` (FormData + token antiforgery) —
   `wwwroot/js/product.js` → `myapp.bindForm`.
2. `Pages/Product/Index.cshtml.cs` → `OnPostCreateAsync`: kiểm quyền hiển thị, `ValidateOnly(Input)`,
   rồi `_session.CallAsync((api, ct) => api.Product.CreateAsync(new CreateProductRequestVM{...}, ct))`.
3. `NodeApiSession` gắn bearer từ cookie phiên, gọi `INodeApi.Product` (typed client), gặp 401
   thì refresh token đúng một lần rồi gọi lại.
4. `ProductControllerImplement.CreateAsync` → `_api.Build().WithUrlPostJson("api/product", model).ExecuteApiAsync<ProductVM>()`.
5. `ProductController.CreateAsync` (implement `INodeProductController`): kiểm quyền bằng
   `NodePermissions`, map VM → `CreateProductCommand`, gọi `IProductService.CreateAsync`, trả THẲNG
   `ProductVM`; `ApiEnvelopeResultFilter` bọc thành `ApiResponse<T>`.
6. `ProductService`: đọc user từ `IHttpContextAccessor`, kiểm luật nghiệp vụ, gọi
   `IProductRepository.CreateAsync(new NewProductRecordVM(...))`, map `ProductRecordVM` nhận về
   sang `ProductVM` (điền cờ theo người xem) — service không thấy Dbo.
7. `ProductRepository.CreateAsync`: map `NewProductRecordVM` → `ProductDbo` + hàng quyền của
   chủ trong một transaction, `SaveChangesAsync`, map Dbo → `ProductRecordVM` rồi trả về.
8. Lỗi ở bất kỳ tầng nào: service ném `ServiceException`/trả `DeletionResult`; controller đổi thành
   `ApiErrorException`; `ExceptionWrapperMiddleware` ghi `ApiErrorResponse{RequestId, Error{Code, Message}}`
   đúng mã HTTP; client typed ném `ApiException<ApiErrorResponse>`; PageModel/`NodeSessionExceptionFilter`
   đổi thành `PageJson.Fail(...)`; JS hiện toast theo `message` + gắn `errors` vào ô.

---

## 4. Quy ước từng tầng

### 4.1 Entity (`<P>.Database.Entities`)

```csharp
public sealed class ProductDbo
{
    public Guid Id { get; set; } = Guid.NewGuid();          // khoá chính Guid; int identity chỉ cho bảng danh mục
    [MaxLength(255)] public required string Name { get; set; }
    [MaxLength(1000)] public string? Description { get; set; }
    public required Guid OwnerUserId { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow; // MỌI thời gian là UTC (xem asp.md)
    public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? DeletedAt { get; set; }

    public ICollection<ProductPermissionDbo> Permissions { get; set; } = new List<ProductPermissionDbo>();
    public LocalUserDbo? Owner { get; set; }
}
```

- Hậu tố `Dbo`, `sealed`, `required` cho cột bắt buộc, nullable cho cột tuỳ chọn.
- Attribute DUY NHẤT được phép: `[MaxLength]` (BCL) — EF đọc nó, giao diện cũng đọc nó để đặt `maxlength`
  ô nhập; **Configuration KHÔNG gọi `HasMaxLength`** nữa, không thì hai nơi lệch. Trần riêng của ô nhập
  ngắn hơn cột thì dùng attribute riêng (`[InputMaxLength]`).
- Không logic nghiệp vụ trong entity (một `GetUri()` thuần tính toán thì được).
- Trạng thái dùng enum ở `Contracts`; nếu lưu chuỗi mã dây thì `HasConversion` ở Configuration.

### 4.2 Configuration, DbContext, migration (`<P>.Database.<Provider>`)

```csharp
public sealed class ProductConfiguration : IEntityTypeConfiguration<ProductDbo>
{
    public void Configure(EntityTypeBuilder<ProductDbo> builder)
    {
        builder.ToTable("Products");                       // PascalCase, số nhiều = tên DbSet
        builder.HasKey(x => x.Id);
        builder.Property(x => x.Name).IsRequired();          // KHÔNG HasColumnName: cột = tên property
        builder.Property(x => x.Status).HasDefaultValue(ProductStatus.Active); // enum: default là giá trị CÓ THẬT
        builder.HasIndex(x => x.OwnerUserId);
        builder.HasOne(x => x.Owner).WithMany(x => x.Products).HasForeignKey(x => x.OwnerUserId).OnDelete(DeleteBehavior.Restrict);
    }
}
```

- Một file Configuration cho một entity, `ApplyConfigurationsFromAssembly` trong `OnModelCreating`.
- `DbContext`: `public DbSet<ProductDbo> Products => Set<ProductDbo>();` (expression-bodied, tên =
  tên bảng). Ghi doc comment cho `DbSet` nào có ý nghĩa không hiển nhiên.
- Kiểm tra xuyên suốt (độ dài cột, UTC) đặt ở override `SaveChanges(bool)`/`SaveChangesAsync(bool, ct)`
  hoặc `ValueConverter` gắn trong `OnModelCreating` — **không** rải ở từng repository.
- `IDesignTimeDbContextFactory` đọc `efdesign.json` + `efdesign.{Environment}.json` (file thứ hai
  `.gitignore`), base path `AppContext.BaseDirectory`, copy-to-output nhưng `CopyToPublishDirectory=Never`.
  KHÔNG đặt tên `appsettings.json` (trùng với host lúc publish, NETSDK1152). Chi tiết ở `database-EF.md`.
- `dotnet ef migrations add X --project src/<app>/<P>.Database.<Provider> --startup-project <chính nó>`.
  Tên migration PascalCase động từ: `AddProductRolePermissions`, `DropUserSystemPrompt`.
- **Đúng MỘT host gọi `MigrateAsync`** (`<Provider>SchemaInitializer.InitializeAsync`); host khác chỉ
  `GetPendingMigrationsAsync` + log cảnh báo. Seed tĩnh (`HasData`) KHÔNG dùng `DateTime.UtcNow`.
- Migration đã apply/push thì không sửa — chồng migration mới.

### 4.3 Repository (`Repositories.Abstractions` + `Repositories`)

```csharp
public interface IProductRepository
{
    Task<ProductRecordVM?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<IReadOnlyList<ProductRecordVM>> GetAllForUserAsync(Guid userId, CancellationToken ct = default);
    Task<int> GetCountAsync(CancellationToken ct = default);
    Task<ProductRecordVM> CreateAsync(NewProductRecordVM product, CancellationToken ct = default);
    Task<bool> RenameAsync(Guid id, string name, CancellationToken ct = default); // false = không có sản phẩm
    Task DeleteAsync(Guid id, CancellationToken ct = default);     // xoá mềm — ghi rõ trong doc comment
    Task HardDeleteAsync(Guid id, CancellationToken ct = default); // xoá hẳn kèm con — ghi rõ những gì KHÔNG dọn
}

public sealed class ProductRepository : IProductRepository
{
    private readonly NodeDbContext _db;
    public ProductRepository(NodeDbContext db) => _db = db;

    public async Task<ProductRecordVM?> GetByIdAsync(Guid id, CancellationToken ct = default)
        => await _db.Products.AsNoTracking()
            .Where(k => k.Id == id && k.DeletedAt == null)
            .Select(ProductMapping.ToRecordExpression) // Expression<Func<ProductDbo, ProductRecordVM>>
            .FirstOrDefaultAsync(ct);

    public async Task<bool> RenameAsync(Guid id, string name, CancellationToken ct = default)
    {
        var row = await _db.Products.FirstOrDefaultAsync(k => k.Id == id && k.DeletedAt == null, ct);
        if (row is null) return false;
        row.Name = name;                             // chỉ đúng cột của ý định này
        await _db.SaveChangesAsync(ct);
        return true;
    }
}
```

- Một repository cho một entity gốc (aggregate); bảng con thuần (permission, event) vẫn có repo riêng
  nếu service cần đọc/ghi độc lập.
- **Dbo chỉ sống bên trong `Repositories`**: không đi vào (tham số) cũng không đi ra (kiểu trả về) khỏi
  repository. Chữ ký chỉ có **VM riêng của repository** (`<X>RecordVM`, `New<X>RecordVM`), kiểu BCL,
  enum/hằng của Contracts — **không** Dbo, không `IQueryable`, không `Expression<Func<>>`, và **không**
  VM của API (`ProductVM`, `ProductListItemVM`, `CreateProductRequestVM`…).
  - VM riêng của repository nằm trong `<P>.ViewModels/Records/` (namespace `<P>.ViewModels.Records`),
    tách khỏi VM của API:
    - `<X>RecordVM` là các cột của bảng cộng cột nối cần thiết (tên tệp, đường dẫn…), không có navigation.
    - `New<X>RecordVM` là input của lệnh tạo.
    - Điều BẮT BUỘC duy nhất là Dbo không đi vào/ra khỏi Repositories; chọn kiểu nào còn lại là chuyện
      cho gọn. Không nhất thiết trả VM, tham số cũng không nhất thiết gói thành VM: kiểu BCL đủ nghĩa thì dùng thẳng (`int`, `bool`, `IReadOnlyList<string>`,
      `IReadOnlyDictionary<string, decimal>` cho tổng theo khoá…). Chỉ dựng RecordVM khi kết quả có
      nhiều trường cần tên; kết quả gom kiểu đó đặt tên theo nghĩa, vẫn hậu tố `RecordVM`
      (vd `BillingTotalRecordVM`).
  - Service map RecordVM → VM của API và điền các cờ theo người xem (`IsFavorite`, `CanManage`…).
    Repository không bao giờ trả thẳng VM của API, kể cả khi hình dạng khớp. Lý do:
    - VM của API có trường tính theo người xem mà repository không biết.
    - Service thường cần những cột mà API không được thấy (`Status`, `DeletedAt`, đường dẫn tệp, hash).
    - Tách hai loại thì đổi hợp đồng API không kéo theo đổi repository.
  - RecordVM có thể mang dữ liệu nội bộ (hash, đường dẫn trên đĩa). Vì vậy controller **không bao giờ
    trả RecordVM ra API**: chỉ trả VM của API.
  - Đọc: chiếu thẳng ra RecordVM bằng `Select(XMapping.ToRecordExpression)` + `AsNoTracking()`, không nạp
    cả entity rồi mới map.
    - Navigation tuỳ chọn viết `x.Nav == null ? null : x.Nav.Prop`, để EF và bản `Compile()` của fake
      chạy giống nhau.
    - `!` chỉ dùng cho khoá ngoại bắt buộc.
  - Ghi: repository map `New<X>RecordVM` → Dbo bên trong. Hàm map để ở `Repositories/Mappings/<X>Mapping.cs`:
    - `internal static class` gồm `ToRecordExpression`, và ngay dưới nó
      `private static readonly Func<> _toRecord = ToRecordExpression.Compile()`. Hai dòng phải đứng
      liền nhau vì thứ tự khởi tạo field static.
    - Thêm extension `ToRecord()` và `ToDbo()`.
    - Định dạng cột JSON (đọc lẫn ghi) cũng nằm ở đây.
    - Mapping chỉ được dùng Entities/ViewModels/Contracts/BCL, không EF: fake trong `<P>.Testing` chép
      nguồn các file này vào (`<Compile Include=... Link=...>`) để map giống hệt bản thật.
  - Lệnh sửa cột chuỗi đi qua nạp hàng + gán + `SaveChangesAsync`, không `ExecuteUpdateAsync`. Lý do:
    `SaveChangesAsync` của DbContext soát độ dài cột, còn `ExecuteUpdate` đi vòng qua bước soát đó.
  - `CreateAsync` trả RecordVM thì dựng từ dữ liệu đã có. Nếu phải đọc lại sau `SaveChangesAsync` thì
    đọc bằng `CancellationToken.None`: lệnh ghi đã commit rồi, huỷ ở bước đọc sẽ làm nơi gọi tưởng ghi
    hỏng và chạy nhánh bù trừ.
  - Lý do cả luật: Dbo là hình dạng bảng.
    - Lọt lên service/controller thì tầng trên dính vào schema: đổi một cột là sửa cả cây.
    - Dễ vô tình trả Dbo kèm navigation hoặc cột nhạy cảm ra API.
    - Service sửa một entity đang được theo dõi mà không biết rằng lần `SaveChangesAsync` kế tiếp (của
      bất kỳ repository nào dùng chung DbContext) sẽ ghi luôn nó.
- **Lệnh sửa viết theo ý định**, không có `UpdateAsync(entity)` cả bản ghi: mỗi thao tác một method nói
  đúng việc (`RenameAsync(id, name)`, `SetStatusAsync(id, status)`, `ArchiveAsync(id)`). Repository tự nạp
  hàng, gán ĐÚNG các cột của ý định đó rồi `SaveChangesAsync`; trả `bool`/RecordVM để service biết có tìm thấy
  không. Lý do: service không cầm Dbo nên không còn "entity sửa xong đưa lại"; và chỉ ghi đúng cột thì
  hai thao tác đồng thời trên hai cột khác nhau không đè nhau.
- Mỗi method **tự `SaveChangesAsync`**; repository không có `SaveChangesAsync()` công khai và không có
  Unit of Work — service không được ghép nhiều lệnh ghi thành một transaction từ bên ngoài. Bất biến
  nhiều bước (dựng sản phẩm + ghi quyền + gán cột) thì gom vào **một method repository** chạy trong transaction
  qua `CreateExecutionStrategy().ExecuteAsync(...)` (bắt buộc khi `EnableRetryOnFailure`), và ghi lý do ở
  doc comment.
- Đặt tên: `GetByIdAsync`, `GetBy<Field>Async`, `GetAllAsync`, `GetAll<Điều kiện>Async`,
  `Get<Aggregate>WithXAsync` (RecordVM kèm dữ liệu con), `ExistsAsync`, `GetCountAsync`, `CreateAsync`/`AddAsync`
  (nhận `New<X>RecordVM`, trả `<X>RecordVM` vừa tạo), `AddRangeAsync`, lệnh sửa theo ý định (`RenameAsync`,
  `Set<Field>Async`, động từ nghiệp vụ như `ArchiveAsync`), `UpsertAsync` (chỉ cho bản ghi kiểu thiết lập
  một-hàng-theo-khoá), `DeleteAsync` (mềm), `HardDeleteAsync` (cứng), `Ensure<X>Async` (tạo nếu chưa có,
  trả id đang dùng).
- Truy vấn thô (full-text, health) tách interface riêng (`IFullTextSearchService`) vẫn nằm ở
  `Repositories.Abstractions`, cài đặt bằng ADO.NET ở `Repositories`.
- Đăng ký DI: `services.AddScoped<IProductRepository, ProductRepository>()` ở host, trong
  `<App>ServiceCollectionExtensions.Add<App>Services(configuration)` cùng chỗ `AddDbContext`.

### 4.4 Service (`Services.<Feature>`)

```
MyApp.Node.Services.Product/
├── AssemblyMarker.cs                    # public sealed class ProductAssemblyMarker; — mốc quét validator
├── ServiceCollectionExtensions.cs       # public static IServiceCollection AddProductServices(this IServiceCollection)
└── Product/
    ├── ProductService.cs
    ├── PersonalProductService.cs
    ├── Interfaces/IProductService.cs  # interface chỉ feature này dùng
    ├── Commands/CreateProductCommand.cs   # record + AbstractValidator<T> (FluentValidation)
    ├── Models/…                         # kiểu trả về riêng của feature (nếu có)
    └── Validators/…                     # hoặc để validator cạnh command
```

- `sealed class XService : IXService`, mọi phụ thuộc qua constructor, field `private readonly _camelCase`.
- Input: `Command` (record `required`/`init` + validator) cho lệnh ghi; VM request dùng thẳng được khi
  không có luật kiểm riêng. Output: VM của `ViewModels`. Service **không bao giờ thấy Dbo** (project
  không tham chiếu `Database.Entities`): nhận RecordVM từ repository, map sang VM của API, gọi lệnh ghi theo ý định, trả VM
  cho controller. Controller cũng vậy — cần dữ liệu thì đi qua service, không tự gọi repository để cầm hàng.
- Lỗi nghiệp vụ: (a) trả record kết quả (`DeletionResult.Ok/NotFound/Conflict(code, message)`) khi "không
  làm được" là câu trả lời bình thường; (b) ném `ServiceException.NotFound/Conflict/BadRequest(code, message)`
  (mang sẵn `StatusCode` + `Code`) cho nhánh hiếm. Service **không** tham chiếu `Api.Core` nên không ném
  `ApiErrorException`; middleware của host dịch `ServiceException` → HTTP.
- Kiểm QUYỀN không nằm trong service nghiệp vụ mà ở `Rules/<X>AccessService` (Services.Common) và được
  controller gọi trước; service nhận `userId`/`roles` như tham số, doc comment ghi rõ "KHÔNG kiểm quyền ở đây".
- Truy vấn trong vòng lặp: lấy MỘT lượt cả danh sách rồi tra từ điển; lọc trước vòng lặp.
- Log bằng structured message tiếng Việt: `_logger.LogInformation("Đã xoá sản phẩm {ProductId} ({Name}).", id, name)`.
- Mã lỗi nghiệp vụ là hằng `UPPER_SNAKE` gom ở `Common/Errors/<Group>Errors.cs` (`AuthErrors.InvalidCredentials`).

### 4.5 Facade `<P>.Services` và DI

```csharp
public static class AppServiceCollectionExtensions
{
    private static readonly Assembly[] FeatureAssemblies =
    [
        typeof(CommonAssemblyMarker).Assembly,
        typeof(ProductAssemblyMarker).Assembly,
        // thêm feature mới: thêm mốc Ở ĐÂY và lời gọi Add<Feature>Services() Ở DƯỚI
    ];

    public static IServiceCollection AddApplicationServices(this IServiceCollection services)
    {
        services.AddValidatorsFromAssemblies(FeatureAssemblies);
        services.AddProductServices();
        return services;
    }
}
```

- Host: `Program.cs` gọi `AddApplicationServices()` (facade) + `Add<App>Services(configuration)` (host:
  DbContext, repositories, client typed, cache, hosted service, mail…). Mọi `AddScoped/AddSingleton` có
  lifetime KHÁC mặc định phải kèm comment lý do (queue singleton vì hai request phải nhìn cùng một sổ;
  cache snapshot singleton; typed HttpClient luôn transient).
- Test `ServiceRegistrationTests` (xunit) so `FeatureAssemblies` với mọi assembly `<P>.Services.*.dll` và
  so mọi `Add<Feature>Services` có được facade gọi — bỏ sót mốc là gãy CI chứ không nổ lúc chạy.
- Hosted service nằm ở project riêng `<P>.BackgroundServices` (đứng trên facade vì một service nền hay
  chạm nhiều feature cùng lúc) — **KHÔNG** đặt trong facade, trong `Services.<Feature>` hay trong host Api.
  Feature chỉ giữ phần việc (`UsageOutboxFlusher` đăng ký `AddScoped` ở feature), cái đồng hồ gọi nó
  (`UsageOutboxBackgroundService`) nằm ở project nền — feature không tham chiếu ngược lên được.
  - Đăng ký: `AddHostedService<>` viết ở host (`Add<App>Services`), đặt NGAY CẠNH singleton/scoped mà
    service đó cần, kèm comment lý do; KHÔNG gom thành một extension trong project nền. Tiện ích
    chỉ-Debug thì bọc `#if DEBUG` ở `Program.cs`.
  - Khuôn viết: `sealed class … : BackgroundService` + `IServiceScopeFactory` (repo/client là scoped →
    mỗi nhịp `CreateScope()`), chu kỳ đọc từ `IConfiguration` có mặc định, mỗi vòng lặp bọc `try/catch`
    riêng — vòng hỏng không được giết dịch vụ; `OperationCanceledException` khi `stoppingToken` huỷ thì
    thoát êm. Tách thân một nhịp ra hàm `public` (`SweepAsync(ct)`) để test gọi thẳng đúng một nhịp;
    `tests/<P>.Tests` tham chiếu project nền.
  - Tiến trình host khác (Windows service, `ServiceHost`) giữ hosted service của riêng nó, không dồn về đây.

### 4.6 API host (`<P>.Api`)

```csharp
[ApiController]
[Route("api/product")]
[Authorize]
public sealed class ProductController : ControllerBase, INodeProductController
{
    [HttpGet("{productId}")]
    public async Task<ProductVM> GetAsync(Guid productId, CancellationToken cancellationToken = default)
    {
        await EnsurePermissionAsync(productId, NodePermissions.ProductView, cancellationToken);
        return await _productService.GetByIdAsync(productId, cancellationToken)
            ?? throw ApiErrorException.NotFound("PRODUCT_NOT_FOUND", "Không tìm thấy sản phẩm.");
    }

    [HttpDelete("{productId}")]
    public async Task DeleteAsync(Guid productId, CancellationToken cancellationToken = default)
    {
        await EnsurePermissionAsync(productId, NodePermissions.ProductManage, cancellationToken);
        (await _productService.DeleteAsync(productId, cancellationToken)).EnsureSuccess();
    }
}
```

- Controller **implement interface của `Integration.Api`** → chữ ký server/client khớp nhau do compiler
  bắt. Action trả THẲNG VM (`Task<T>` / `Task`), không `IActionResult`, không `Ok(...)`/`NotFound(...)`;
  lỗi ném `ApiErrorException.BadRequest/Forbidden/NotFound/Conflict(code, message, details?)`.
- Upload: tham số `Stream content` + `[FromFormFile]`, `string fileName` + `[FromFormFileName]`, JSON kèm
  trong form + `[FromFormJson("optionsJson")]`, thân thô `[FromRawBody]` (đều ở `Api.Core/ModelBinding`).
  Trả file: action trả `Task<Stream>` + `Response.SetFileDownload(contentType, fileName)`. SSE:
  `Response.StartServerSentEventsAsync()` rồi ghi thẳng. Endpoint không muốn bọc vỏ: `[RawResponse]`.
- `Program.cs` theo thứ tự: Serilog (console + file rolling, template file có ngày đầy đủ) → JWT
  (`OnTokenValidated` tra DB thu hồi phiên + thay claim vai bằng vai thật; `OnChallenge` trả
  `ApiErrorResponse` mã `TOKEN_REVOKED`) → `AddAuthorization` (policy hỏi QUYỀN, không `RequireRole` tên
  vai) → `AddOpenApi` + Scalar (chỉ Development) → `AddApplicationServices` + `Add<App>Services` →
  `AddControllers(o => { o.Filters.Add<ApiEnvelopeResultFilter>(); o.ModelBinderProviders.Insert(0, …); })
  .AddNewtonsoftJson(o => <P>JsonSettings.Apply(o.SerializerSettings))
  .ConfigureApiBehaviorOptions(o => o.InvalidModelStateResponseFactory = … ApiErrorResponse "VALIDATION_FAILED")`
  → CORS → build → `UseRequestId` → `UseSerilogRequestLogging` → `UseExceptionWrapper` → OpenAPI →
  `UseCors` → `UseAuthentication` → `UseAuthorization` → `MapControllers` → `InitializeDatabaseAsync`
  (bọc try/catch + `Log.Fatal` + `CloseAndFlushAsync` — dưới IIS lỗi ở đây chỉ là trang 500.30 trắng) → `Run`.
- `ExceptionWrapperMiddleware`: `switch` exception → status (`ApiErrorException`, `ServiceException`,
  `AuthException` → 401, `FieldTooLongException` → 400 kèm `field`, còn lại 500 `INTERNAL_ERROR` + `traceId`);
  4xx log `Warning`, 5xx log `Error`; kiểm `Response.HasStarted` trước khi ghi.
- Cấu hình bắt buộc (`ConnectionStrings`, `Jwt:SigningKey`, `Central:BaseUrl`…) **KHÔNG có giá trị mặc
  định** — thiếu thì ném `InvalidOperationException` lúc khởi động với tên khoá, không lùi về localhost.
- `Properties/launchSettings.json` phải khai `ASPNETCORE_ENVIRONMENT=Development` cho từng profile; cổng
  cố định, ghi vào bản đồ cổng của project.

### 4.7 Client typed (`<P>.Integration.Api`) — dùng bộ tích hợp ASP + thư viện HTTP nội bộ (tên gói ở `local.md`)

```csharp
public interface INodeApi : IBaseAddress
{
    INodeAuthController Auth { get; }
    INodeProductController Product { get; }   // mỗi controller server = một property
    void SetBearerToken(string? token);
}

/// <summary>Nhóm <c>api/product</c>.</summary>
public interface INodeProductController
{
    /// <summary><c>GET api/product/{productId}</c>.</summary>
    Task<ProductVM> GetAsync(Guid productId, CancellationToken cancellationToken = default);
}

public sealed partial class NodeApiImplement : MyAppApiBase, INodeApi
{
    public NodeApiImplement(IHttpClientFactory<NodeApiImplement> f) : base(f.CreateClient())
    {
        Product = new ProductControllerImplement(this);
    }
    public INodeProductController Product { get; }
}

// file NodeApiImplement.ProductControllerImplement.cs
public sealed partial class NodeApiImplement
{
    sealed class ProductControllerImplement : INodeProductController
    {
        const string BaseUrl = "api/product";
        readonly NodeApiImplement _api;
        public ProductControllerImplement(NodeApiImplement api) => _api = api;

        public Task<ProductVM> GetAsync(Guid productId, CancellationToken cancellationToken = default)
            => _api.Build().WithUrlGet($"{BaseUrl}/{productId}").ExecuteApiAsync<ProductVM>(cancellationToken);
    }
}
```

- Một partial file cho mỗi nhóm: `<App>ApiImplement.<Group>ControllerImplement.cs`. Doc comment của mỗi
  method interface ghi `METHOD đường/dẫn` để đọc interface là biết endpoint.
- `ExecuteApiAsync<T>` bóc `ApiResponse<T>.Data` (null → ném), `ExecuteApiVoidAsync` cho `Task`,
  `ExecuteApiResponseAsync<T>` khi cần `Meta`/`RequestId`. Tham số query: `new UrlBuilder(BaseUrl).WithParamIfNotNull(...)`;
  đoạn đường dẫn từ chuỗi người dùng thì `Uri.EscapeDataString`.
- `<P>ApiBase : BaseApi` đặt `DefaultJsonSerializerSettings = <P>JsonSettings.Create()` (resolver mặc
  định của thư viện HTTP nội bộ BỎ QUA chuỗi rỗng → "đặt về rỗng" thành "không đổi").
- Đăng ký: `services.AddApiHttpClientFactory(); services.AddApiHttpClient<I<App>Api, <App>ApiImplement>(configureClient)`,
  trả `IHttpClientBuilder` để gắn `DelegatingHandler` (chuyển tiếp User-Agent/IP, gắn danh tính node).
  **KHÔNG** `ConfigurePrimaryHttpMessageHandler` bật `UseCookies`: handler dùng chung cả tiến trình →
  cookie phiên của người sau đè người trước. Danh tính đi theo request (bearer qua `SetBearerToken`,
  client là transient) hoặc thân request.

### 4.8 VM (`<P>.ViewModels`)

- `public sealed record ProductVM(Guid Id, string Name, ..., bool IsPersonal = false);` — positional
  record, tham số có ý nghĩa không hiển nhiên ghi doc comment ngay trên tham số. Request dùng record
  `{ required init }`. Hậu tố: `VM` (đọc), `RequestVM` (thân ghi), `ListItemVM` (dòng danh sách),
  `ResultVM`. Nhóm nhiều VM nhỏ cùng feature vào một file `<Feature>VMs.cs` được.
- Cùng một định nghĩa dùng ở CẢ controller lẫn client → đổi trường là compiler bắt cả hai đầu.
- Tên trường trên dây = camelCase của tên property (resolver chung); chỉ `[JsonProperty("x")]` khi bắt
  buộc khớp hệ ngoài. Enum lên dây là **chuỗi** (`StringEnumConverter`), enum có mã dây riêng thì
  `[JsonConverter]` gắn trên enum type ở `Contracts` + `IModelBinderProvider` ở `Api.Core`.
- Cờ "được làm gì" (`CanManage`, `CanTransferOwner`) trả trong VM để Web ẨN nút — không phải để canh
  cửa; mỗi hành động vẫn qua chốt riêng ở API.

### 4.9 Web host (Razor Pages, `<P>.Web`)

```
Pages/
├── _ViewImports.cshtml         # @inject ILocalizerService L, @addTagHelper *, <P>.Web
├── _ViewStart.cshtml           # Layout = "_Layout"
├── Shared/_Layout.cshtml, _Icons.cshtml (sprite SVG inline), _Pager.cshtml, _UserMenu.cshtml…
├── Auth/Login.cshtml(.cs), Logout, ForgotPassword, ResetPassword, TwoFactor
├── <Feature>/Index.cshtml(.cs), Detail.cshtml(.cs)
│   ├── _IndexContent.cshtml    # thân trang, dùng lại cho mọi lối vào
│   ├── _<X>List.cshtml         # partial danh sách: render lần đầu VÀ trả về qua ?handler=List
│   └── _<X>Modals.cshtml
└── Dashboard/<Feature>/Index.cshtml(.cs)   # khu quản trị, khoá cả thư mục bằng AuthorizeFolder
Infrastructure/
├── <App>ApiSession.cs          # bọc I<App>Api: bearer từ cookie, 401 → refresh một lần → gọi lại; SemaphoreSlim cho lời gọi song song
├── PageJson.cs                 # Ok/Fail/Invalid/Fragment/FromApiException + ValidateOnly(this PageModel, model, prefix)
├── PartialRenderer.cs          # render partial → string cho JSON
├── PagerModel.cs               # phân trang, khoá query là "p" (KHÔNG BAO GIỜ "page")
├── <App>Auth.cs                # tên cookie, policy, StoreTokens, SignOutAsync, hằng thời hạn
├── <App>SessionExceptionFilter.cs / <App>SessionExpiredException.cs
├── <App>PermissionsMiddleware.cs # hỏi API quyền của người gọi MỖI request, gắn vào HttpContext.User (cookie chỉ giữ danh tính)
├── StatusBadge.cs, Breadcrumb.cs, RequestExtensions.cs (IsAjax, ReturnUrlFor)
wwwroot/
├── css/site.css, js/site.js    # myapp.* dùng chung: submitForm, postJson, getJson, filterList, bindForm, confirmTwice, bulkBar, t()
├── js/<page>.js                # một file mỗi trang, gói trong IIFE, export init() để shell gọi lại sau khi đổ HTML
└── lib/<thư viện>/             # thư viện nhúng sẵn, commit, cache immutable
```

- PageModel `sealed class IndexModel : PageModel`, inject `<App>ApiSession` + `PartialRenderer`
  (+ `ILocalizerService`). Tham số lọc: `[BindProperty(SupportsGet = true, Name = "q")] string? Search`.
  Form: `[BindProperty] CreateInput Input` với `sealed class CreateInput` lồng trong PageModel,
  DataAnnotations `[Required(ErrorMessage = "...")]` + `[FieldLimit("Product.Name")]` (độ dài lấy từ
  API `api/field-limits`, Web không tự khai số).
- Handler: `OnGetAsync` (render), `OnGet<X>Async` trả `PageJson.Ok(new { html })` cho lọc/phân trang,
  `OnPost<X>Async` trả `PageJson.Ok(data, message)` / `PageJson.Fail(message, code)` /
  `PageJson.Invalid(ModelState)` sau `this.ValidateOnly(Input, nameof(Input))`. **Không** `return Page()`
  ở POST. Bấm nút quyết định ở JS (`confirmTwice`), server không hỏi lại. Lượt hàng loạt: đi tuần tự,
  nuốt `ApiException` từng mục, trả `{ deleted, failed: [{ id, message }] }`.
- `Program.cs` theo thứ tự: Serilog → đọc URL API (thiếu thì ném) → `AddHttpContextAccessor` →
  `Add<App>Api(client => BaseAddress/Timeout).AddHttpMessageHandler<ClientContextForwardingHandler>()`
  → `AddScoped<<App>ApiSession>()`, `PartialRenderer`, cache singleton → cookie auth (`LoginPath`
  `/auth/login`, `SameSite=Lax`, `SecurePolicy=SameAsRequest`, ba sự kiện `OnRedirectToReturnUrl`/
  `OnRedirectToLogin`/`OnRedirectToAccessDenied` trả JSON 200/401/403 khi `IsAjax()`) → `AddAuthorization`
  (policy `RequireAssertion(ctx => ctx.User.Can(Permission))`) → `AddAntiforgery(HeaderName)` →
  `AddRazorPages(o => { AuthorizeFolder("/"); AuthorizeFolder("/Dashboard", policy); AllowAnonymousToPage("/Auth/Login")… })
  .AddMvcOptions(o => { Filters.Add<SessionExceptionFilter>(); Filters.Add(new AutoValidateAntiforgeryTokenAttribute()); })
  .AddNewtonsoftJson(...)` → (YARP chỉ cho SSE/file/zip, gắn bearer qua `ProxyAccessTokenAsync`, xoá header
  `Cookie`) → build → `UseSerilogRequestLogging` → `UseExceptionHandler("/error")` + `UseHsts` (ngoài Dev) →
  `/health/live`, `/health/ready` → `UseStaticFiles` (lib/ cache immutable) → `UseRequestLocalization` →
  `WarmUpJsonLocalizers` → `UseRouting` → `UseAuthentication` → `Use<App>Permissions` → `UseAuthorization`
  → `MapRazorPages` → `MapReverseProxy` → `Run`.
- Static file: `<link href="~/css/site.css" asp-append-version="true">`; script trang trong
  `@section Scripts`. Icon: sprite SVG inline `<svg class="bi"><use href="#bi-x"/></svg>`, không font icon.
- Mọi luật Razor/AJAX/ModelState/`page`/`model` chi tiết ở `asp.md` — đọc trước khi viết trang.

### 4.10 Vỏ JSON, lỗi, requestId (`Contracts.Common` + `Api.Core`)

```json
{ "requestId": "req_…", "data": { … }, "meta": { "page": 1, "pageSize": 20, "total": 57, "totalPages": 3 } }
{ "requestId": "req_…", "error": { "code": "PRODUCT_NOT_FOUND", "message": "Không tìm thấy sản phẩm.", "details": { "field": "Name" } } }
```

- `RequestIdMiddleware`: đọc/sinh `X-Request-Id`, đặt vào `HttpContext.Items["RequestId"]` và header
  phản hồi; mọi log lỗi và thân lỗi đều mang nó.
- `<P>JsonSettings.Create()/Apply()` là nguồn DUY NHẤT của quy ước JSON nội bộ (camelCase +
  `StringEnumConverter` + resolver riêng). JSON của bên thứ ba tự khai settings riêng — không dùng chung.
- Web nhúng JSON vào trang: `PageJson.Embed(value)` (đi qua cùng settings), không `JsonConvert.SerializeObject` trần.

---

## 5. Đa ngôn ngữ (i18n) — project `<P>.Resources`

```
<P>.Resources/
├── <P>Cultures.cs            # Vietnamese = "vi", English = "en" (TRUNG TÍNH, không vi-VN), Data = "en-US", Default = vi, SupportedUiCultures, IsSupported()
├── LocalizerService.cs       # ILocalizerService { IStringLocalizer<NavResource> Nav; IStringLocalizer<AuthResource> Auth; … } + LocalizerService (primary ctor)
├── JsonLocalizerWarmup.cs    # WarmUpJsonLocalizers(IServiceProvider, cultures): nạp mọi (resource × culture) lúc khởi động — vá bug thread-safety của JsonLocalizer 4.0.x
└── Resources/
    ├── NavResource.cs        # public sealed class NavResource; + static class NavResourceExtensions { public static string Dashboard(this IStringLocalizer<NavResource> l) => l["Nav.Dashboard"]; }
    ├── NavResource.vi.json   # { "Nav.Dashboard": "Tổng quan", … }
    ├── NavResource.en.json
    ├── JsResource.cs         # ToBrowserDictionary(): mọi khoá "Js.*" → từ điển cho JS
    └── …                     # SharedResource, AuthResource, ErrorResource, StatusResource, <Feature>Resource
```

- Gói `Askmethat.Aspnet.JsonLocalizer` (ghim 4.0.1, xem comment trong `Directory.Packages.props`),
  `LocalizationMode.I18n`, `ResourcesPath = "Resources"`. csproj: `<None Update="Resources\**\*.json"
  CopyToOutputDirectory="PreserveNewest">` — JsonLocalizer đọc từ ĐĨA, thiếu dòng này build vẫn xanh
  nhưng mọi chuỗi hiện đúng tên khoá.
- Chế độ I18n gộp MỌI file cùng ngôn ngữ vào một từ điển ⇒ **khoá bắt buộc mang tiền tố nhóm**
  (`Nav.Dashboard`, `Auth.EmailRequired`, `Js.LogoutTitle`); trùng khoá giữa hai file là đè nhau âm thầm.
- Mỗi nhóm có một class marker `<Group>Resource` + extension method **có tên** cho từng khoá (gõ
  `L.Nav.Dashboard()` — compiler bắt khoá sai, `L["Nav.Dashbord"]` thì không). Chuỗi có tham số:
  `l["Node.PagerItems", count]`. Tra theo giá trị thô (trạng thái từ API): `<Group>.<giá trị>`
  (`Status.dns_pending`), dùng `LocalizedString.ResourceNotFound` để phân biệt "chưa dịch" và trả `null`
  cho nơi gọi hiện giá trị thô.
- `Program.cs` (Web): `AddMemoryCache(); AddJsonLocalization(o => { ResourcesPath; CacheDuration; LocalizationMode = I18n; DefaultCulture })`;
  `AddScoped<ILocalizerService, LocalizerService>()`; `AddRazorPages().AddViewLocalization().AddDataAnnotationsLocalization()`;
  `UseRequestLocalization(new RequestLocalizationOptions { DefaultRequestCulture = new(Data, Default), SupportedCultures = DataCultures, SupportedUICultures = SupportedUiCultures, RequestCultureProviders = [Cookie, AcceptLanguage] })`
  đặt **trước** `UseAuthentication` (sự kiện cookie auth cũng tra localizer) → `app.Services.WarmUpJsonLocalizers(...)`.
- **Culture (đọc/ghi số, ngày) ghim `en-US` cho mọi người; chỉ UICulture đổi.** Thả Culture theo ngôn ngữ
  thì ô nhập giá `1.5` parse ra hai giá trị khác nhau tuỳ người.
- Đổi ngôn ngữ: trang `Pages/SetLanguage.cshtml(.cs)` — `OnPost(culture, returnUrl)` kiểm `IsSupported`,
  ghi cookie `CookieRequestCultureProvider.DefaultCookieName` (1 năm, `Secure = Request.IsHttps`, `SameSite=Lax`),
  `Url.IsLocalUrl(returnUrl)` rồi redirect; `OnGet` → `Redirect("/")`; `AllowAnonymousToPage("/SetLanguage")`.
  Partial `_LanguageSwitcher.cshtml` dùng `<form method="post" asp-page="/SetLanguage">` (KHÔNG `action=`,
  mất token antiforgery → 400), `returnUrl = Request.Path + Request.QueryString`. Tên ngôn ngữ luôn viết
  bằng chính ngôn ngữ đó.
- `_ViewImports.cshtml`: `@using <P>.Resources`, `@using <P>.Resources.Resources` (namespace extension
  method — thiếu là "không tìm thấy phương thức" dù inject được), `@inject ILocalizerService L`.
  Trong `.cshtml`: `@L.Nav.Dashboard()`; trong PageModel: inject `ILocalizerService _localizer`.
- JS: partial `_I18nScript.cshtml` đổ `L.Js.ToBrowserDictionary()` (Newtonsoft,
  `StringEscapeHandling.EscapeHtml`) vào `window.__myappI18n` + `window.__myappCulture`; `site.js` có
  `myapp.t(key, args)`. Chỉ chuỗi JS thật sự dựng lúc chạy (toast, hộp xác nhận) vào nhóm `Js.*` — mỗi khoá
  là byte gửi kèm mọi trang.
- DataAnnotations: `[Required(ErrorMessage = "Auth.EmailRequired")]`, `[Display(Name = "Vm.FullName")]` —
  `AddDataAnnotationsLocalization()` coi `ErrorMessage` là KHOÁ. Chỉ attribute BCL (Required, Range,
  StringLength, RegularExpression, EmailAddress…) được dịch tự động; **`ValidationAttribute` tự viết phải
  tự tra `IStringLocalizer` trong `IsValid(object?, ValidationContext)`**, không thì người dùng thấy
  nguyên khoá trên form.
- API trả lỗi bằng `Code` (UPPER_SNAKE) + `Message` tiếng Việt; Web dịch theo `Code` (`Errors.<Code>`) khi
  có khoá, lùi về `Message` của API khi chưa có. Cố ý KHÔNG dịch: tên hãng/sản phẩm, mã giao thức, tên
  trường của dịch vụ ngoài, đơn vị kỹ thuật (`tokens`).
- Hai project Web trong cùng repo dùng chung một `<P>.Resources` khi chuỗi trùng nhau đủ nhiều; không thì
  mỗi Web một project Resources riêng — không nhét thư mục `Resources/` vào host.

---

## 6. Quy tắc đặt tên — dạng lạc đà (PascalCase / camelCase), KHÔNG snake_case, KHÔNG kebab-case

| Thứ | Quy tắc | Ví dụ |
|---|---|---|
| Project / assembly / namespace | `<Prefix>.<Area>.<Layer>[.<Feature>]`, namespace = đường dẫn thư mục | `MyApp.Node.Services.Product` |
| class / record / struct / interface / enum / delegate | PascalCase; interface `I` đầu; mỗi type một file cùng tên | `ProductService`, `IProductRepository` |
| Hậu tố theo vai | `Dbo`, `Configuration`, `Repository`, `Mapping` (Dbo ↔ RecordVM, nội bộ Repositories), `Service`, `Controller`, `ControllerImplement`, `Command`, `CommandValidator`, `VM`/`RequestVM`/`ListItemVM` (API), `RecordVM`/`New<X>RecordVM` (repository), `Resource`, `Middleware`, `BackgroundService`/`Service` (nền), `AssemblyMarker`, `Options`, `Exception`, `Errors` (hằng mã lỗi), `Extensions` | `CreateProductCommandValidator`, `NavResourceExtensions` |
| Method | PascalCase, bất đồng bộ **luôn** `…Async` và nhận `CancellationToken` cuối cùng | `GetByIdWithPermissionsAsync` |
| Property / event / const / static readonly | PascalCase | `MaxRetryCount`, `FeatureAssemblies` |
| Field private | `_camelCase` | `_productRepo`, `_tokenLock` |
| Tham số / biến cục bộ | camelCase | `productId`, `cancellationToken` (KHÔNG `ct` ở chữ ký public; `ct` chỉ trong lambda/private) |
| Giá trị enum | PascalCase; mã dây gắn bằng attribute | `GoogleVertexAi` ↔ `"google_vertex_ai"` |
| Handler Razor | `OnGet<Tên>Async`, `OnPost<Tên>Async`; `asp-page-handler="Tên"` | `OnPostBulkDeleteAsync` |
| File `.cs` / `.cshtml` | = tên type / tên trang, PascalCase; partial `_PascalCase.cshtml`; partial class `<Cha>.<Con>.cs` | `_ProductList.cshtml`, `DocumentIngestStarter.StagedSource.cs` |
| File `.js` / `.css` trong `wwwroot` | camelCase theo trang | `productDetail.js`, `site.js` |
| **Bảng DB** | PascalCase, **số nhiều**, = tên `DbSet` | `Products`, `ProductPermissions`, `AppUsers` |
| **Cột DB** | PascalCase, **= đúng tên property C#**, KHÔNG `HasColumnName` | `OwnerUserId`, `CreatedAt`, `MaxBatchSize` |
| Khoá chính / khoá ngoại | `Id`; `<Entity>Id` | `ProductId`, `OwnerUserId` |
| Index / FK / unique | để EF tự đặt (`IX_…`, `FK_…`, `AK_…`); chỉ `HasDatabaseName` khi cần tên ổn định cho SQL tay | — |
| Migration | PascalCase động từ + đối tượng | `AddDocumentSourceFileHash` |
| Mã lỗi nghiệp vụ, mã quyền | `UPPER_SNAKE` (là DỮ LIỆU trên dây, không phải định danh C#) | `PRODUCT_NOT_FOUND`, `product.manage` (quyền: chấm) |
| Khoá i18n | `<Nhóm>.<Khoá>` PascalCase | `Auth.EmailRequired` |
| Route URL | chữ thường, đoạn nhiều từ nối bằng `-` (đây là HTTP, không phải C#) | `api/product/{productId}/role-permissions` |
| Trường JSON trên dây | camelCase (resolver chung tự đổi) | `ownerUserId` |
| Test method | tên tiếng Việt PascalCase không dấu cách, mô tả hành vi | `SảnPhẩmNhápKhôngRaDanhSáchChung` |

Ghi chú DB:
- Cột PascalCase trên PostgreSQL: EF tự bọc nháy kép nên code không phải bận tâm, nhưng **SQL gõ tay
  phải viết `"MaxBatchSize"`** — không nháy thì Postgres hạ chữ thường và báo cột không tồn tại.
- Bảng cũ đang snake_case (di sản của repo cũ) thì **giữ nguyên**, chỉ cột/bảng THÊM MỚI theo luật này; không
  đi đổi tên hàng loạt (đụng mọi bản đã deploy). Muốn đổi thì hỏi người dùng (xem đầu file).
- Cột thời gian hậu tố `At` (`CreatedAt`, `DeletedAt`), luôn UTC; cột cờ tiền tố `Is`/`Has`/`Can`;
  cột đếm hậu tố `Count`; tiền hậu tố đơn vị (`CostUsd`, `SizeBytes`, `TimeoutSeconds`).

Namespace theo kind trong feature (`Interfaces/`, `Models/`, `Enums/`, `Extensions/`, `Helpers/`,
`Commands/`, `Validators/`) — luật đầy đủ ở `csharp.md`.

---

## 7. Checklist thêm một tính năng CRUD mới (theo đúng thứ tự, mỗi bước build xanh rồi mới bước tiếp)

1. `Database.Entities/<X>Dbo.cs` (+ navigation ở entity cha).
2. `Database.<Provider>/Configurations/<X>Configuration.cs`; thêm `DbSet` vào DbContext.
3. `dotnet ef migrations add Add<X>` (startup = chính project Database); sửa tay `defaultValue` enum
   nếu có; kiểm script idempotent nếu là SQL Server.
4. `ViewModels/<Feature>/<X>VMs.cs` (VM, RequestVM, ListItemVM của API) và `ViewModels/Records/<X>RecordVM.cs`, `New<X>RecordVM.cs` — phải có trước vì chữ ký repository
   dùng chúng.
5. `Repositories.Abstractions/Interfaces/I<X>Repository.cs` (chỉ RecordVM, không Dbo, không VM của API) →
   `Repositories/Mappings/<X>Mapping.cs` → `Repositories/<X>Repository.cs` → `AddScoped` ở `Add<App>Services`.
6. `tests/<P>.Testing/Fakes/InMemory<App>Db.cs` thêm `List<<X>Dbo>`;
   `Fakes/Repositories/InMemory<X>Repository.cs` (mô phỏng đúng bộ lọc xoá mềm và cách map ra VM của bản thật).
7. `Services.<Feature>/<Feature>/Commands/<Verb><X>Command.cs` + validator;
   `Interfaces/I<X>Service.cs`; `<X>Service.cs`; đăng ký ở `Add<Feature>Services`. Feature mới hoàn toàn:
   thêm project + `AssemblyMarker` + mốc trong `FeatureAssemblies` + lời gọi trong `AddApplicationServices`.
8. `tests/<P>.Tests/Services/Db/<X>ServiceTests.cs` với `InMemory<App>Db`.
9. `Integration.Api/Interfaces/I<App><X>Controller.cs` (doc `METHOD url`), property trên `I<App>Api`,
   `Implements/<App>ApiImplement.<X>ControllerImplement.cs`, gán trong constructor.
10. `Api/Controllers/<X>Controller.cs : ControllerBase, I<App><X>Controller` — kiểm quyền từng action.
11. `Resources/Resources/<X>Resource.cs` + `.vi.json` + `.en.json` (Web có i18n).
12. `Web/Pages/<Feature>/Index.cshtml(.cs)` + `_IndexContent` + `_<X>List` (+ `_<X>Modals`),
    `wwwroot/js/<feature>.js` (`init()` + `myapp.filterList` + `myapp.bindForm`), thêm mục điều hướng,
    `AuthorizeFolder`/policy nếu là khu quản trị.
13. `tests/<P>.Tests/Web/…` cho logic thuần của PageModel (nhãn, điều hướng, sắp xếp).
14. Cập nhật `docs/` (glossary/thuật ngữ nếu có) và memory project nếu có quyết định không đọc được từ code.

---

## 8. Những chỗ HAY SAI khi làm theo khuôn này (chi tiết ở file tương ứng)

- Service `Include`/LINQ trực tiếp vì "tiện" → vi phạm §2, chuyển vào repository.
- Quên mốc `AssemblyMarker` khi thêm project feature → validator không đăng ký, không lỗi biên dịch;
  `ServiceRegistrationTests` là chốt.
- Viết `BackgroundService` mới vào facade/feature/host Api cho "gần code" → sai §2; đặt ở
  `<P>.BackgroundServices`, còn `AddHostedService<>` ở host (§4.5). Quên dòng đăng ký thì service không
  bao giờ chạy mà không có lỗi nào báo.
- Controller `return NotFound()`/`Ok(...)` → không khớp interface client; dùng `ApiErrorException`.
- Cột mới đặt snake_case cho "giống bảng cũ" → sai luật §6; `migrations remove` rồi sinh lại.
- `Database.<Provider>` có file tên `appsettings.json` → trùng với host lúc publish (NETSDK1152).
- Hai host cùng `MigrateAsync` → `PK___EFMigrationsHistory` trùng khoá lúc bật cùng lúc (`database-EF.md`).
- Web `AddHttpClient` với `CookieContainer` → hai người dùng tráo phiên (§4.7).
- `UseRequestLocalization` đặt sau `UseAuthentication` → sự kiện cookie auth tra localizer ra ngôn ngữ mặc định.
- `[BindProperty] page`/`asp-route-page` → hỏng handler (`asp.md`).
- Gặp project cũ khác khuôn mà tự "sửa luôn cho đúng chuẩn" hoặc tự "làm theo cũ cho đỡ đụng" — cả hai
  đều sai: phải hỏi người dùng (đầu file).

# Clean Architecture Foundation — BookingTicket

Tài liệu này ghi lại các quyết định kiến trúc ở giai đoạn **Foundation**. Chưa có nghiệp vụ đặt vé, EF Core, JWT hay migrations được triển khai.

## 1. Mục tiêu

Tách hệ thống thành 4 assembly với hướng phụ thuộc luôn **đi vào trong**:

```text
BookingTicket (API / Presentation)
        │
        ├──────────────► BookingTicket.Application
        │                       │
        └──────────────► BookingTicket.Infrastructure
                                │
                                ├──► BookingTicket.Application
                                └──► BookingTicket.Domain

BookingTicket.Application ──────► BookingTicket.Domain
BookingTicket.Domain ───────────► (không phụ thuộc project nào)
```

| Project | Vai trò | Phụ thuộc |
|---|---|---|
| `BookingTicket` (`BookingTicket.csproj`) | Presentation / Composition Root | Application, Infrastructure |
| `BookingTicket.Domain` | Domain Layer | Không có |
| `BookingTicket.Application` | Application Layer | Domain |
| `BookingTicket.Infrastructure` | Infrastructure Layer | Application, Domain |

## 2. Vì sao tách 4 project

- **Biên dịch bắt hướng phụ thuộc.** Nếu Domain `using Infrastructure`, compiler sẽ cho phép — trừ khi tách assembly và cấm reference. Project reference là cơ chế thực thi kiến trúc, không chỉ là quy ước.
- **Domain sống độc lập.** Luật đặt vé, trạng thái vé, ghế, suất chiếu không cần biết SQL Server hay EF Core.
- **Use case không biết transport.** Application xử lý "đặt vé" bằng interface, không biết request đến từ HTTP hay job nền.
- **Chi tiết kỹ thuật nằm ngoài cùng.** EF Core, JWT, Azure SQL, email... nằm ở Infrastructure/API và có thể thay đổi mà không sửa Domain.
- **API là nơi ghép (Composition Root).** API đăng ký implementation cụ thể vào DI container; các layer trong không tự khởi tạo dependency của mình.

## 3. Trách nhiệm từng layer

### Domain
- Entity, Value Object, Domain Event, rule nghiệp vụ cốt lõi.
- Interface domain thuần (nếu thực sự cần, ví dụ repository interface dùng bởi Domain service).
- Không reference `Microsoft.EntityFrameworkCore`, ASP.NET, JWT, SDK Azure.

### Application
- Use case / Application service: điều phối một hành động nghiệp vụ.
- DTO/Command/Query, validator, mapping.
- **Định nghĩa interface cổng** (ví dụ `IBookingRepository`, `IUnitOfWork`, `ITokenService`) — đây là abstraction theo nhu cầu của use case, không tạo sẵn abstraction "phòng khi".

### Infrastructure
- Implement interface do Application định nghĩa.
- EF Core DbContext, repository implementation, migrations, external clients, JWT signing thực tế.
- Reference Application để implement interface; reference Domain để dùng entity.

### API
- Controller, middleware, DI registration, cấu hình appsettings, auth pipeline.
- Chứa `WeatherForecastController` hiện tại (giữ nguyên).
- Không chứa logic nghiệp vụ; controller chỉ nhận request và gọi use case.

## 4. Vì sao Domain không được phụ thuộc Infrastructure

Nếu Domain reference Infrastructure:

- Luật nghiệp vụ bị gắn với DB/EF/JWT → không test unit được nếu không có hạ tầng.
- Không thể đổi Azure SQL sang nguồn dữ liệu khác mà không đụng code rule.
- Dependency cycle rất dễ xuất hiện: Infrastructure cần Domain, Domain lại cần Infrastructure.

Nguyên tắc **Dependency Inversion**: abstraction thuộc về layer sử dụng nó (Application), implementation thuộc về layer ngoài (Infrastructure). Domain vì thế không cần biết abstraction hay implementation hạ tầng.

## 5. Project Reference có ý nghĩa gì

`ProjectReference` tạo ra dependency biên dịch và runtime:

- MSBuild build đúng thứ tự: Domain → Application → Infrastructure → API.
- Layer trong **không thể** `using` layer ngoài vì không có reference.
- Visual Studio / Rider hiển thị đúng biểu đồ dependency.
- Một reference sai hướng là lỗi kiến trúc ngay tại compile time.

Lưu ý đặc biệt: vì project API nằm ở thư mục gốc và `src/` nằm bên trong, SDK Web mặc định quét `**/*.cs`, bao gồm cả `src/*/obj/**/AssemblyInfo.cs`, gây lỗi `CS0579`. Đã khắc phục trong `BookingTicket.csproj` bằng:

```xml
<DefaultItemExcludes>$(DefaultItemExcludes);src\**</DefaultItemExcludes>
```

`DefaultItemExcludes` (thay vì chỉ `Compile Remove`) loại trừ `src/` khỏi tất cả default glob của Web SDK, nên cũng tránh copy file build của project con khi publish.

## 6. Cấu trúc thư mục

### Đang có ở giai đoạn Foundation

```text
BookingTicket/
├── Controllers/                  # giữ nguyên WeatherForecastController
├── Properties/                   # giữ nguyên launchSettings.json
├── appsettings.json              # giữ nguyên
├── appsettings.Development.json  # giữ nguyên
├── Program.cs                    # giữ nguyên
├── WeatherForecast.cs            # giữ nguyên
└── src/
    ├── BookingTicket.Domain/
    ├── BookingTicket.Application/
    └── BookingTicket.Infrastructure/
```

Ba file `Class1.cs` là artifact của `dotnet new classlib`; được giữ nguyên theo yêu cầu không xóa code hiện tại. Chúng không có logic và sẽ được thay thế bằng file có ý nghĩa khi triển khai nghiệp vụ đầu tiên.

### Sẽ tạo khi có use case đầu tiên (không tạo sẵn lúc này)

```text
src/BookingTicket.Domain/
├── Entities/          # Movie, Showtime, Seat, Booking...
├── ValueObjects/      # Money, SeatNumber...
└── Events/            # BookingConfirmed...

src/BookingTicket.Application/
├── Abstractions/      # interface cổng do use case cần
├── Bookings/          # Command/Query/Handler cho đặt vé
└── DTOs/

src/BookingTicket.Infrastructure/
└── Persistence/       # DbContext, Repository implementation, Migrations
```

Mỗi thư mục chỉ được tạo khi có file đầu tiên đặt vào đó.

## 7. Ví dụ luồng request đặt vé (chưa triển khai)

```text
Next.js  POST /api/bookings  { showtimeId, seatIds }
   │
   ▼
API Controller            → đọc request, gọi handler
   │
   ▼
Application               → handler kiểm tra command, dùng IBookingRepository
   │                         (interface do Application định nghĩa)
   ▼
Domain                    → rule: ghế còn trống, không trùng ghế, giá vé hợp lệ
   │
   ▼
Infrastructure            → implementation IBookingRepository ghi qua EF Core
   │                         vào Azure SQL, commit transaction
   ▼
API                       → trả 201 / DTO về Next.js
```

Điểm mấu chốt: Application gọi interface, API inject implementation. Domain không bao giờ biết EF Core hay SQL Server đang được dùng.

## 8. Kiểm tra

```bash
dotnet sln BookingTicket.slnx list
dotnet list BookingTicket.csproj reference
dotnet restore BookingTicket.slnx
dotnet build BookingTicket.slnx
```

Kết quả kỳ vọng: 4 project trong solution, reference đúng hướng, build `0 Warning(s), 0 Error(s)`.

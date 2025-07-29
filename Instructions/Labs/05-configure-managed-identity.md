---
lab:
  title: Azure SQL Database에 대한 관리 ID 구성
  module: Explore Azure SQL Database safety practices for development
---

# Azure SQL Database에 대한 관리 ID 구성

이 연습에서는 코드에 자격 증명을 저장하지 않고 샘플 웹앱에 관리 ID를 구성합니다.

Azure App Service는 확장성이 뛰어나고 자체 유지 관리되는 웹 호스팅 솔루션을 제공합니다. 주요 기능 중 하나는 애플리케이션에 대한 관리 ID를 제공하여 Azure SQL 데이터베이스 및 기타 Azure 서비스에 대한 액세스 보안을 간소화하는 것입니다. 관리 ID를 사용하면 연결 문자열에 자격 증명과 같은 중요한 정보를 저장할 필요가 없어 앱의 보안을 강화할 수 있습니다. 

이 연습을 완료하는 데 약 **30**분 정도 소요됩니다.

## 시작하기 전에

이 연습을 시작하기 전에 다음을 수행해야 합니다.

- 리소스를 만들고 관리할 수 있는 적절한 권한이 있는 Azure 구독.
- 컴퓨터에 설치된 [**SSMS(SQL Server Management Studio)**](https://learn.microsoft.com/en-us/ssms/install/install)
- [**Visual Studio Code**](https://code.visualstudio.com/download?azure-portal=true)가 컴퓨터에 설치되어 있고 다음 확장자가 설치되어 있어야 합니다.
    - [Azure App Service](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-azureappservice?azure-portal=true)

## 웹 애플리케이션 및 Azure SQL 데이터베이스 만들기

먼저 웹 애플리케이션 및 Azure SQL 데이터베이스를 만듭니다.

1. [Azure Portal](https://portal.azure.com?azure-portal=true)에 로그인합니다.
1. **구독**을 검색하여 선택합니다.
1. **설정**에서 **리소스 공급자**로 이동하여 **Microsoft.Sql** 공급자를 검색한 다음 **등록**을 선택합니다.
1. Azure Portal의 메인 페이지로 돌아가서 **리소스 만들기**를 선택합니다.
1. **웹앱**을 검색하고 선택합니다.
1. **만들기**를 선택하고 필요한 세부 정보를 입력합니다.

    | 그룹 | 설정 | 값 |
    | --- | --- | --- |
    | **프로젝트 세부 정보** | **구독** | Azure 구독을 선택합니다. |
    | **프로젝트 세부 정보** | **리소스 그룹** | 리소스 그룹 선택 또는 새로 만들기 |
    | **인스턴스 세부 정보** | **이름** | 웹앱에 대한 고유한 이름을 입력합니다. |
    | **인스턴스 세부 정보** | **런타임 스택** | .NET 8(LTS) |
    | **인스턴스 세부 정보** | **지역** | 웹앱을 호스팅할 지역을 선택합니다. |
    | **가격 책정 계획** | **요금제** | 기본 |
    | **데이터베이스** | **엔진** | SQLAzure |
    | **데이터베이스** | **서버 이름** | SQL 서버의 고유 이름을 입력합니다. |
    | **데이터베이스** | **데이터베이스 이름** | 데이터베이스의 고유한 이름을 입력합니다. |
    

    > **참고:** 프로덕션 워크로드의 경우 **표준 - 범용 프로덕션 앱**을 선택합니다. 새 데이터베이스의 사용자 이름과 비밀번호가 자동으로 생성됩니다. 배포 후 이러한 값을 검색하려면 앱의 **환경 변수** 페이지에 있는 **연결 문자열**로 이동합니다. 

1. **검토 + 만들기**를 선택한 다음, **만들기**를 선택합니다. 배포를 완료하는 데 몇 분 정도 걸릴 수 있습니다.
1. SSMS에서 데이터베이스에 연결하고 다음 코드를 실행합니다.

    >**팁**: 웹앱 리소스의 서비스 커넥터 페이지에서 연결 문자열을 확인하여 서버에 할당된 사용자 ID 및 암호를 가져올 수 있습니다.
 
    ```sql
    CREATE TABLE Products (
        ProductID INT PRIMARY KEY,
        ProductName NVARCHAR(100),
        Category NVARCHAR(50),
        Price DECIMAL(10, 2),
        Stock INT
    );
    
    INSERT INTO Products (ProductID, ProductName, Category, Price, Stock) VALUES
    (1, 'Laptop', 'Electronics', 999.99, 50),
    (2, 'Smartphone', 'Electronics', 699.99, 150),
    (3, 'Desk Chair', 'Furniture', 89.99, 200),
    (4, 'Coffee Maker', 'Appliances', 49.99, 100),
    (5, 'Book', 'Books', 19.99, 300);
    ```

## SQL 관리자로 계정 추가

다음으로, 데이터베이스에 대한 계정 액세스를 추가합니다. 이 연습의 후속 단계를 위한 필수 조건인 다른 Microsoft Entra ID 사용자를 만들 수 있는 계정은 Microsoft Entra를 통해 인증된 계정만 만들 수 있기 때문에 이 작업이 필요합니다.

1. 이전에 만든 Azure SQL 서버로 이동합니다.
1. **설정** 왼쪽 메뉴에서 **Microsoft Entra ID**를 선택합니다.
1. **관리자 설정**을 선택합니다.
1. 계정을 검색하고 선택합니다.
1. **저장**을 선택합니다.

## 관리 ID 사용

다음으로, 자동화된 자격 증명 관리를 허용하는 보안 모범 사례인 Azure 웹앱에 대해 시스템이 할당한 관리 ID를 사용하도록 설정합니다.

1. Azure Portal에서 해당 웹앱으로 이동합니다.
1. 왼쪽 메뉴의 **설정**에서 **ID**를 선택합니다.
1. **시스템 할당** 탭에서 **상태**를 **켜기**로 전환하고 **저장**을 선택합니다. 웹앱에 대해 시스템이 할당한 관리 ID를 사용할지 묻는 메시지가 표시되면 **예**를 선택합니다.

## Azure SQL 데이터베이스에 액세스 허용

1. SSMS를 사용하여 Azure SQL 데이터베이스에 다시 연결합니다. **Microsoft Entra MFA**를 선택하고 사용자 이름을 제공합니다.
1. 데이터베이스를 선택한 다음 새 쿼리 편집기를 엽니다.
1. 다음 SQL 명령을 실행하여 관리 ID에 대한 사용자를 만들고 필요한 권한을 할당합니다. 웹앱 이름을 제공하여 스크립트를 편집합니다.

    ```sql
    CREATE USER [your-web-app-name] FROM EXTERNAL PROVIDER;
    ALTER ROLE db_datareader ADD MEMBER [your-web-app-name];
    ALTER ROLE db_datawriter ADD MEMBER [your-web-app-name];
    ```

## 웹 애플리케이션 만들기

다음으로, Azure SQL 데이터베이스와 Entity Framework Core를 사용하여 제품 테이블의 제품 목록을 표시하는 ASP.NET 애플리케이션을 만듭니다.

### 프로젝트 만들기

1. VS Code에서 새 폴더를 만듭니다. 폴더 이름을 프로젝트 이름으로 지정합니다.
1. 터미널을 열고 다음 명령을 실행하여 새 MVC 프로젝트를 만듭니다.
    
    ```dos
   dotnet new mvc
    ```
    그러면 선택한 폴더에 새 ASP.NET MVC 프로젝트가 만들어지고 Visual Studio Code에서 로드됩니다.

1. 다음 명령을 실행하여 애플리케이션을 실행합니다. 

    ```dos
   dotnet run
    ```
1. 터미널에 *지금 수신 중: http://localhost:<port>* 이라고 출력됩니다. 웹 브라우저에서 URL로 이동하여 애플리케이션에 액세스합니다. 

1. 웹 브라우저를 닫고 애플리케이션을 중지합니다. 또는 VS Code 터미널에서 `Ctrl+C`을(를) 눌러 애플리케이션을 중지할 수도 있습니다.

### Azure SQL Database에 연결하도록 프로젝트 업데이트

다음으로 관리 ID를 사용하여 Azure SQL 데이터베이스에 성공적으로 연결할 수 있는 몇 가지 구성을 업데이트합니다.

1. 프로젝트에서 SQL Server에 필요한 NuGet 패키지를 추가합니다.
    ```dos
    dotnet add package Microsoft.EntityFrameworkCore.SqlServer
    ```
1. 프로젝트의 루트 폴더에서 **appsettings.json** 파일을 열고 `ConnectionStrings` 섹션을 삽입합니다. 여기서 `<server-name>` 및 `<db-name>`을(를) 서버와 데이터베이스의 실제 이름으로 바꿉니다. 이 연결 문자열은 `Models/MyDbContext.cs` 파일의 기본 생성자가 데이터베이스에 대한 연결을 설정하는 데 사용됩니다.

    ```json
    {
      "Logging": {
        "LogLevel": {
          "Default": "Information",
          "Microsoft.AspNetCore": "Warning"
        }
      },
      "AllowedHosts": "*",
      "ConnectionStrings": {
        "DefaultConnection": "Server=<server-name>.database.windows.net,1433;Initial Catalog=<db-name>;Authentication=Active Directory Default;"
      }
    }
    ```
1. 파일을 저장 후 닫습니다.

### 코드 추가

1. 프로젝트의 **모델** 폴더에서 다음 코드를 사용하여 제품 엔터티에 대한 **Product.cs** 파일을 만듭니다. `<app name>`을(를) 애플리케이션의 실제 이름으로 바꿉니다.

    ```csharp
    namespace <app name>.Models;
    
    public class Product
    {
        public int ProductId { get; set; }
        public string ProductName { get; set; }
        public string Category { get; set; }
        public decimal Price { get; set; }
        public int Stock { get; set; }
    }
    ```
1. 프로젝트의 루트 폴더에 **Database** 폴더를 만듭니다.
1. 프로젝트의 **Database** 폴더에서 다음 코드를 사용하여 제품 엔터티에 대한 **MyDbContext.cs** 파일을 만듭니다. `<app name>`을(를) 애플리케이션의 실제 이름으로 바꿉니다.

    ```csharp
    using <app name>.Models;
    
    namespace <app name>.Database;
    
    using Microsoft.EntityFrameworkCore;
    
    public class MyDbContext : DbContext
    {
        public MyDbContext(DbContextOptions<MyDbContext> options) : base(options)
        {
        }
    
        public DbSet<Product> Products { get; set; }
    }    
    ```
1. 프로젝트의 **Controllers** 폴더에서 **HomeController.cs** 파일의 `HomeController` 및 `IActionResult` 클래스를 편집하고 다음 코드를 사용하여 `_context` 변수를 추가합니다.

    ```csharp
    private MyDbContext _context;

    public HomeController(ILogger<HomeController> logger, MyDbContext context)
    {
        _logger = logger;
        _context = context;
    }

    public IActionResult Index()
    {
        var data = _context.Products.ToList();
        return View(data);
    }
    ```

1. 또한 파일의 맨 위에 `using.<app name>.Database`를 추가합니다.
1. 프로젝트의 **보기 -> 홈** 폴더에서 **Index.cshtml** 파일을 업데이트하고 다음 코드를 추가합니다.

    ```html
    <table class="table">
        <thead>
            <tr>
                <th>Product Id</th>
                <th>Product Name</th>
                <th>Category</th>
                <th>Price</th>
                <th>Stock</th>
            </tr>
        </thead>
        <tbody>
            @foreach(var item in Model)
            {
                <tr>
                    <td>@item.ProductId</td>
                    <td>@item.ProductName</td>
                    <td>@item.Category</td>
                    <td>@item.Price</td>
                    <td>@item.Stock</td>
                </tr>
            }
        </tbody>
    </table>
    ```

1. **Program.cs** 파일을 편집하고 제공된 코드 조각을 `var app = builder.Build();` 줄 바로 위에 삽입합니다. 이렇게 변경하면 애플리케이션의 시작 시퀀스 중에 코드가 실행됩니다. `<app name>`을(를) 애플리케이션의 실제 이름으로 바꿉니다.

    ```csharp
    using Microsoft.EntityFrameworkCore;
    using <app name>.Database;

    var builder = WebApplication.CreateBuilder(args);

    // Add services to the container.
    builder.Services.AddControllersWithViews();
    builder.Services.AddDbContext<MyDbContext>(options =>
        options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
    ```

    > **참고:** 배포하기 전에 애플리케이션을 실행하려면 SQL 사용자 자격 증명으로 연결 문자열을 업데이트합니다. 새 데이터베이스의 사용자 이름과 비밀번호가 자동으로 생성되었습니다. 배포 후 이러한 값을 검색하려면 앱의 **환경 변수** 페이지에 있는 **연결 문자열**로 이동합니다. 애플리케이션이 예상대로 실행되고 있는지 확인한 후에는 안전한 배포 프로세스를 위해 관리 ID를 사용하도록 다시 전환합니다.

### 코드 배포

1. `Ctrl+Shift+P`을(를) 눌러 **명령 팔레트**를 엽니다.
1. **Azure App Service: Deploy to Web App...** 을 입력하고 선택합니다.
1. 웹 애플리케이션 코드가 들어 있는 폴더를 선택합니다.
1. 이전 단계에서 만든 웹앱을 선택합니다.
    > 참고: "배포에 필요한 구성이 '앱'에서 누락되었습니다"라는 메시지를 받을 수 있습니다. **구성 추가**를 선택합니다. 그런 다음 지침에 따라 구독 및 앱 서비스 리소스를 선택합니다.
1. 메시지가 표시되면 배포를 확인합니다.

## 애플리케이션 테스트

웹 애플리케이션을 실행하고 저장된 자격 증명 없이 Azure SQL Database에 연결할 수 있는지 확인합니다.

1. 브라우저를 열고 Azure 웹앱의 URL(예: https://your-web-app-name.azurewebsites.net))로 이동합니다.
1. 웹 애플리케이션이 실행 중이고 액세스할 수 있는지 확인합니다.
1. 아래와 같은 웹 페이지가 표시됩니다.

    ![배포 후 웹 애플리케이션의 스크린샷.](./Media/01-app-page.png)

## 지속적인 배포 설정(선택)

1. `Ctrl+Shift+P`을(를) 눌러 **명령 팔레트**를 엽니다.
1. **Azure App Service: Configure Continuous Delivery...** 를 입력하고 선택합니다.
1. 프롬프트에 따라 GitHub 리포지토리 또는 Azure DevOps에서 지속적인 배포를 설정합니다.

**시스템이 할당한 관리 ID** 대신 **사용자가 할당한 관리 ID**를 사용하는 것이 유리할 수 있는 시나리오를 고려합니다.

## 정리

본인 소유의 구독으로 이 모듈을 진행하고 있는 경우에는 프로젝트가 끝날 때 여기서 만든 리소스가 계속 필요한지 확인하는 것이 좋습니다. 

리소스를 불필요하게 실행하면 추가 비용이 발생할 수 있습니다. 리소스를 개별적으로 삭제하거나 [Azure Portal](https://portal.azure.com?azure-portal=true)에서 전체 리소스 집합을 삭제할 수 있습니다.

## 자세한 정보

Azure SQL 데이터베이스의 자동 장애 조치(failover) 그룹에 대한 자세한 내용은 [Azure SQL용 Microsoft Entra의 관리 ID](https://learn.microsoft.com/azure/azure-sql/database/authentication-azure-ad-user-assigned-managed-identity?azure-portal=true)를 참조합니다.

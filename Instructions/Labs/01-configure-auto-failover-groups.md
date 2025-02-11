---
lab:
  title: Azure SQL 데이터베이스에 대한 자동 장애 조치(failover) 그룹으로 애플리케이션 복원력 사용
  module: Get started with Azure SQL Database for cloud-native application development
---

# Azure SQL 데이터베이스에 대한 자동 장애 조치(failover) 그룹으로 애플리케이션 복원력 사용

이 연습에서는 주 데이터베이스와 보조 데이터베이스 역할을 하는 두 개의 Azure SQL 데이터베이스를 만듭니다. 애플리케이션 데이터베이스의 고가용성 및 재해 복구를 보장하고 애플리케이션에서 복제 상태의 유효성을 검사하도록 자동 장애 조치(failover) 그룹을 구성합니다.

이 연습을 완료하는 데 약 **30**분 정도 소요됩니다.

## 시작하기 전에

이 연습을 시작하기 전에 다음을 수행해야 합니다.

- 리소스를 만들고 관리할 수 있는 적절한 권한이 있는 Azure 구독.
- [**Visual Studio Code**](https://code.visualstudio.com/download?azure-portal=true)가 컴퓨터에 설치되어 있고 다음 확장자가 설치되어 있어야 합니다.
    - [C# 개발 키트](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csdevkit?azure-portal=true)

## 기본 및 보조 Azure SQL 서버 만들기

먼저 주 서버와 보조 서버를 모두 설정하고 **AdventureWorksLT** 샘플 데이터베이스를 사용하겠습니다.

1. [Azure Portal](https://portal.azure.com?azure-portal=true)에 로그인합니다.

1. Azure Portal 오른쪽 상단에 있는 Cloud Shell 아이콘을 선택합니다. `>_` 기호처럼 보입니다. 메시지가 표시되면 셸 유형으로 **Bash**를 선택합니다.

1. Cloud Shell 터미널에서 다음 명령을 실행합니다. `<your_resource_group>`, `<your_primary_server>`, `<your_location>`, `<your_secondary_server>`, `<your_admin_password>` 값을 실제 값으로 바꿉니다.

    * 리소스 그룹 만들기
    ```azurecli
    az group create --name <your_resource_group> --location <your_location>
    ```

    * 기본 SQL Server 만들기
    ```azurecli        
    az sql server create --name <your_primary_server> --resource-group <your_resource_group> --location <your_location> --admin-user "sqladmin" --admin-password <your_admin_password>
    ```

    * 보조 SQL Server를 만듭니다. 동일한 스크립트로, 서버 이름과 위치만 변경합니다.
    ```azurecli
    az sql server create --name <your_secondary_server> --resource-group <your_resource_group> --location <your_location> --admin-user "sqladmin" --admin-password <your_admin_password>    
    ```

    * 주 서버에 지정된 가격 책정 계층으로 샘플 데이터베이스를 만듭니다.
    ```azurecli
    az sql db create --resource-group <your_resource_group> --server <your_primary_server> --name AdventureWorksLT --sample-name AdventureWorksLT --service-objective "S0"    
    ```
    
1. 배포가 완료되면 만든 기본 Azure SQL Server로 이동합니다.
1. 왼쪽 창의 **보안**에서 **네트워킹**을 선택합니다. 방화벽 규칙에 IP 주소를 추가합니다.
1. **Azure 서비스 및 리소스에서 이 서버에 액세스할 수 있도록 허용** 옵션을 선택합니다.
1. **저장**을 선택합니다.
1. 보조 서버에 대해 위의 단계를 반복합니다.

    이러한 단계를 수행하면 구조화되고 중복된 Azure SQL Database 환경을 사용할 수 있습니다.

## 자동 장애 조치(failover) 그룹 구성

다음으로, 이전에 설정한 Azure SQL Database에 대한 자동 장애 조치(failover) 그룹을 만듭니다. 여기에는 두 서버 간에 장애 조치(failover) 그룹을 설정하고 설정이 제대로 작동하는지 확인하는 작업이 포함됩니다.

1. Cloud Shell 터미널에서 다음 명령을 실행합니다. `<your_failover_group>`, `<your_resource_group>`, `<your_primary_server>`, `<your_secondary_server>` 값을 실제 값으로 바꿉니다.

    * 장애 조치 그룹 만들기
    ```azurecli
    az sql failover-group create -n <your_failover_group> -g <your_resource_group> -s <your_primary_server> --partner-server <your_secondary_server> --failover-policy Automatic --grace-period 1 --add-db AdventureWorksLT
    ```

    * 장애 조치(failover) 그룹 확인
    ```azurecli    
    az sql failover-group show -n <your_failover_group> -g <your_resource_group> -s <your_primary_server>
    ```

    > 잠시 시간을 내어 결과와 `partnerServers` 값을 검토합니다. JEA가 중요한 이유는 무엇일까요?

    > 각 파트너 서버 내에서 `role` 특성을 확인하여 현재 서버가 주 서버로 작동하는지 보조 서버로 작동하는지 확인할 수 있습니다. 이 정보는 장애 조치(failover) 그룹의 현재 구성 및 준비 상태를 이해하는 데 중요합니다. 장애 조치(failover) 시나리오 중에 애플리케이션에 미치는 잠재적 영향을 평가하고 고가용성 및 재해 복구를 위해 설정이 올바르게 구성되었는지 확인하는 데 도움이 됩니다.
    
## 애플리케이션 코드와 통합

.NET 애플리케이션을 Azure SQL 데이터베이스 엔드포인트에 연결하려면 다음 단계를 따라야 합니다.

1. Visual Studio Code에서 터미널을 열고 다음 명령을 실행하여 `Microsoft.Data.SqlClient` 패키지를 설치하고 새 .NET 콘솔 애플리케이션을 만듭니다.

    ```bash
    dotnet new console -n AdventureWorksLTApp
    cd AdventureWorksLTApp 
    dotnet add package Microsoft.Data.SqlClient --version 5.2.1
    ```

1. **Visual Studio Code**에서 이전 단계로 생성한 `AdventureWorksLTApp` 폴더를 엽니다.

1. 프로젝트 디렉터리의 루트에 `appsettings.json` 파일을 만듭니다. 이 구성 파일은 데이터베이스 연결 문자열을 저장합니다. 연결 문자열의 `<your_failover_group>` 및 `<your_password>` 값을 실제 세부 정보로 바꾸어야 합니다.

    ```json
    {
      "ConnectionStrings": {
        "FailoverGroupConnection": "Server=tcp:<your_failover_group>.database.windows.net,1433;Initial Catalog=AdventureWorksLT;Persist Security Info=False;User ID=sqladmin;Password=<your_password>;MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"
      }
    }
    ```

1. `.csproj` 파일을 **Visual Studio Code**에서 열고 `</PropertyGroup>` 태그 바로 아래에 다음 내용을 추가합니다.

    ```xml
    <ItemGroup>
        <PackageReference Include="Microsoft.Data.SqlClient" Version="5.0.0" />
        <PackageReference Include="Microsoft.Extensions.Configuration" Version="6.0.0" />
        <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="6.0.0" />
    </ItemGroup>
    ```

    전체 `.csproj` 파일은 다음과 비슷하게 보일 것입니다.

    ```xml
    <Project Sdk="Microsoft.NET.Sdk">

      <PropertyGroup>
        <OutputType>Exe</OutputType>
        <TargetFramework>net8.0</TargetFramework>
        <ImplicitUsings>enable</ImplicitUsings>
        <Nullable>enable</Nullable>
      </PropertyGroup>
    
      <ItemGroup>
        <PackageReference Include="Microsoft.Data.SqlClient" Version="5.0.0" />
        <PackageReference Include="Microsoft.Extensions.Configuration" Version="6.0.0" />
        <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="6.0.0" />
      </ItemGroup>
    
    </Project>
    ```

1. **Visual Studio Code**에서 `Program.cs` 파일을 엽니다. 편집기에서 모든 기존 코드를 아래 제공된 코드로 바꿉니다.

    > **참고:** 잠시 코드를 검토하고 자동 장애 조치(failover) 그룹의 주 서버 및 보조 서버에 대한 정보를 출력하는 방법을 관찰합니다.

    ```csharp
    using System;
    using Microsoft.Data.SqlClient;
    using Microsoft.Extensions.Configuration;
    using System.IO;
    
    namespace AdventureWorksLTApp
    {
        class Program
        {
            static void Main(string[] args)
            {
                var configuration = new ConfigurationBuilder()
                    .SetBasePath(Directory.GetCurrentDirectory())
                    .AddJsonFile("appsettings.json")
                    .Build();
    
                string connectionString = configuration.GetConnectionString("FailoverGroupConnection");
    
                ExecuteQuery(connectionString);
            }
    
            static void ExecuteQuery(string connectionString)
            {
                using (SqlConnection connection = new SqlConnection(connectionString))
                {
                    try
                    {
                        connection.Open();
                        string query = @"
                            SELECT 
                                @@SERVERNAME AS [Primary_Server],
                                partner_server AS [Secondary_Server],
                                partner_database AS [Database],
                                replication_state_desc
                            FROM 
                                sys.dm_geo_replication_link_status";
    
                        using (SqlCommand command = new SqlCommand(query, connection))
                        {
                            using (SqlDataReader reader = command.ExecuteReader())
                            {
                                while (reader.Read())
                                {
                                    Console.WriteLine($"Primary Server: {reader["Primary_Server"]}");
                                    Console.WriteLine($"Secondary Server: {reader["Secondary_Server"]}");
                                    Console.WriteLine($"Database: {reader["Database"]}");
                                    Console.WriteLine($"Replication State: {reader["replication_state_desc"]}");
                                }
                            }
                        }
                    }
                    catch (Exception ex)
                    {
                        Console.WriteLine($"Error executing query: {ex.Message}");
                    }
                    finally
                    {
                        connection.Close();
                    }
                }
            }
        }
    }
    ```

1. 메뉴에서 **실행** > **디버깅 시작**을 선택하거나 **F5** 키를 눌러 코드를 실행합니다. 상단 도구 모음에서 재생 버튼을 선택하여 애플리케이션을 시작할 수도 있습니다.

    > **중요:** *"C# 디버깅을 위한 확장이 없습니다. 마켓플레이스에서 C# 확장을 찾아야 하나요?"* 라는 메시지가 표시되면 **C# 개발 키트** 확장이 설치되어 있는지 확인합니다.

1. 코드를 실행한 후 Visual Studio Code의 **디버그 콘솔** 탭에 출력이 표시됩니다.

    ```
    Primary Server: <your_server_name>
    Secondary Server: <your_server_name>
    Database: AdventureWorksLT
    Replication State: CATCH_UP
    ```
    
    복제 상태 `CATCH_UP`는 데이터베이스가 파트너와 완전히 동기화되어 장애 조치(failover)할 준비가 되었음을 의미합니다. 복제 상태를 모니터링하면 성능 병목 상태를 식별하고 데이터 복제가 효율적으로 수행되도록 할 수 있습니다.

## 보조 지역으로의 장애 조치(failover)

주 Azure SQL Database에서 지역 가동 중단으로 인해 문제가 발생하는 시나리오를 상상해 보세요. 서비스 연속성을 유지하고 가동 중지 시간을 최소화하려면 강제 장애 조치(failover)를 수행하여 애플리케이션을 보조 복제본으로 전환해야 합니다.

강제 장애 조치(failover) 중에는 모든 새 TDS 세션이 자동으로 보조 서버로 다시 라우팅된 다음 주 서버가 됩니다. 가장 좋은 점은 엔드포인트가 동일하게 유지되므로 애플리케이션 연결 문자열을 변경할 필요가 없다는 것입니다.

장애 조치(failover)를 시작하고 애플리케이션을 실행하여 주 서버와 보조 서버의 상태를 확인해 보겠습니다.

1. Azure Portal로 돌아가서 Cloud Shell 터미널의 새 인스턴스를 엽니다. 다음 코드를 실행합니다. `<your_failover_group>`, `<your_resource_group>`, `<your_primary_server>` 값을 실제 값으로 바꿉니다. `--server` 매개 변수 값은 현재 보조값이어야 합니다.

    ```azurecli
    az sql failover-group set-primary --name <your_failover_group> --resource-group <your_resource_group> --server <your_server_name>
    ```

    > **참고**: 이 작업은 몇 분 정도 걸릴 수 있습니다.

1. 장애 조치(failover)가 완료되면 애플리케이션을 다시 실행하여 복제 상태를 확인합니다. 이제 보조 서버가 주 서버를 대신하고 원래 주 서버가 보조 서버가 된 것을 볼 수 있습니다.

주 및 보조 애플리케이션 데이터베이스를 동일한 지역에 배치하려는 이유와 다른 지역을 선택하는 것이 도움이 될 수 있는 경우를 고려합니다.

## 정리

본인 소유의 구독으로 이 모듈을 진행하고 있는 경우에는 프로젝트가 끝날 때 여기에서 만든 리소스가 계속 필요한지 확인하는 것이 좋습니다. 

리소스를 불필요하게 실행하면 추가 비용이 발생할 수 있습니다. 리소스를 개별적으로 삭제하거나 [Azure Portal](https://portal.azure.com?azure-portal=true)에서 전체 리소스 집합을 삭제할 수 있습니다.

## 자세한 정보

Azure SQL 데이터베이스의 자동 장애 조치(failover) 그룹에 대한 자세한 내용은 [장애 조치(failover) 그룹 개요 및 모범 사례(Azure SQL 데이터베이스)](https://learn.microsoft.com/azure/azure-sql/database/failover-group-sql-db?azure-portal=true)를 참조하세요.

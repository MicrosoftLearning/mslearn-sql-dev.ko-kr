---
lab:
  title: Azure SQL 데이터베이스에서 개발을 위한 데이터 가져오기 및 내보내기
  module: Import and export data for development in Azure SQL Database
---

# Azure SQL 데이터베이스에서 개발을 위한 데이터 가져오기 및 내보내기

이 연습에서는 외부 REST 엔드포인트(Azure Static Web App을 사용하여 시뮬레이션)에서 데이터를 가져오고 Azure 함수를 사용하여 데이터를 내보냅니다. 이 랩에서는 데이터 가져오기/내보내기 작업을 처리하기 위한 REST API 및 Azure Functions를 통합하는 데 중점을 두고, 개발 목적으로 Azure SQL 데이터베이스 작업에 대한 실질적인 경험을 제공합니다.

## 필수 조건

이 랩을 시작하기 전에 다음 사항이 준비되어 있는지 확인합니다.

- 리소스를 만들고 관리할 수 있는 권한이 있는 활성 Azure 구독.
- Azure SQL Database, REST API 및 Azure Functions에 대한 기본 지식
- 다음 확장이 설치된 Visual Studio Code:
      - Azure Functions 확장.
- 리포지토리 복제를 위해 Git이 설치되었습니다.
- 데이터베이스를 관리하기 위한 SSMS(SQL Server Management Studio) 또는 Azure Data Studio.

## 환경 설정

이 랩에 필요한 리소스를 설정하는 것부터 시작하겠습니다. Azure SQL 데이터베이스와 데이터 가져오기 및 내보내기에 필요한 도구가 포함되어 있습니다.

### Azure SQL Database 만들기

이 단계를 수행하려면 Azure에서 데이터베이스를 만들어야 합니다.

1. [Azure Portal](https://portal.azure.com?azure-portal=true)에 로그인합니다.
1. Azure Portal에서 **SQL 데이터베이스** 페이지로 이동합니다.
1. **만들기**를 실행합니다.
1. 필수 필드 작성:

    | 설정 | 값 |
    |---|---|
    | 무료 서버리스 제품 | 제품 적용 |
    | 구독 | 구독 |
    | 리소스 그룹 | *리소스 그룹 선택 또는 새로 만들기* |
    | 데이터베이스 이름 | **MyDB** |
    | 서버 | ***새로 만들기** 링크를 선택합니다* |
    | 서버 이름 | *고유한 이름 선택* |
    | 위치 | *위치를 선택합니다* |
    | 인증 방법 | SQL 인증 |
    | 서버 관리자 로그인 | **sqladmin** |
    | 암호 | *보안 암호 입력* |
    | 암호 확인 | *비밀번호를 확인합니다* |

1. **검토 + 만들기**와 **만들기**를 차례로 클릭합니다.
1. 배포가 완료되면 ***Azure SQL Server***(Azure SQL 데이터베이스가 아님)의 **네트워킹** 섹션으로 이동합니다.
    1. 방화벽 규칙에 IP 주소를 추가합니다. 이렇게 하면 SSMS(SQL Server Management Studio) 또는 Azure Data Studio를 사용하여 데이터베이스를 관리할 수 있습니다.
    1. **Azure 서비스 및 리소스에서 이 서버에 액세스할 수 있도록 허용** 확인란을 선택합니다. 이렇게 하면 Azure 함수 앱에서 데이터베이스 서버에 액세스할 수 있습니다.
    1. 변경 내용을 저장합니다.

> [!NOTE]
> 프로덕션 환경에서는 액세스 유형과 액세스를 허용할 위치를 결정해야 합니다. Entra 인증만 선택하면 함수에 약간의 변경 사항이 있을 수 있지만, Azure 함수 앱이 서버에 액세스할 수 있도록 하려면 *이 서버에 대한 Azure 서비스 및 리소스 액세스 허용*을 사용하도록 설정해야 한다는 점에 유의합니다.

### GitHub 리포지토리를 복제합니다.

1. **Visual Studio Code**를 사용하여 ASP.NET 5 API 앱을 만드는 방법을 보여줍니다.

1. GitHub 리포지토리를 복제하고 프로젝트를 준비합니다.

    1. **Visual Studio Code**에서 **Ctrl+Shift+P**(Windows) 또는 **Cmd+Shift+P**(Mac)를 눌러 **명령 팔레트**를 엽니다.
    1. **Git: Clone**를 입력하고 **Git: Clone**을 선택합니다.
    1. 프롬프트에서 다음 URL을 입력하여 리포지토리를 복제합니다.
        ```bash
        https://github.com/MicrosoftLearning/mslearn-sql-dev.git
        ```

    1. 리포지토리를 복제할 대상 폴더를 선택합니다.

### JSON 데이터에 대한 Azure Blob Storage 설정

이제 **employees.json** 파일을 호스팅하도록 **Azure Blob Storage**를 설정하겠습니다. Azure Portal 및 **Visual Studio Code** 내에서 다음 단계를 수행합니다.

Azure Storage 계정을 만드는 것부터 시작하겠습니다.

1. **Azure Portal**에서 **스토리지 계정** 페이지로 이동합니다.
1. **만들기**를 실행합니다.
1. 필수 필드 작성:

    | 설정 | 값 |
    |---|---|
    | 구독 | 구독 |
    | 리소스 그룹 | 리소스 그룹 선택 또는 새로 만들기 |
    | 스토리지 계정 이름 | 전역적으로 고유한 이름을 선택합니다. |
    | 지역 | 가장 가까운 지역 선택 |
    | 주 서비스 | **Azure Blob Storage 또는 Azure Data Lake Storage Gen2** |
    | 성능 | 표준 |
    | 중복
           | LRS(로컬 중복 스토리지) |

1. **검토 + 만들기**와 **만들기**를 차례로 클릭합니다.
1. 스토리지 계정이 생성될 때까지 기다립니다.

이제 계정이 생겼으니 **employees.json**를 Blob Storage에 업로드해 보겠습니다.

1. Azure Portal에서 **스토리지 계정** 페이지로 이동합니다.
1. 사용자의 스토리지 계정을 선택합니다.
1. **컨테이너** 섹션으로 이동합니다.
1. **jsonfiles**라는 이름의 새 컨테이너를 만듭니다.
1. 컨테이너 내에서 **업로드**를 클릭하고 복제된 디렉토리의 **/Allfiles/Labs/04/blob-storage** 아래에 있는 **employees.json** 파일을 업로드합니다.

파일에 대한 익명 액세스를 허용할 수도 있지만, 이 경우에는 이 파일에 대해 *SAS(공유 액세스 서명)* 을 생성하여 안전한 액세스를 보장하겠습니다.

1. **jsonfiles** 컨테이너에서 **employees.json** 파일을 선택합니다.
1. 파일의 컨텍스트 메뉴에서 **SAS 생성**을 선택합니다.
1. 설정을 검토하고 **SAS 및 URL 생성**을 선택합니다.
1. Blob SAS 토큰 및 Blob SAS URL이 생성됩니다. 다음 단계에서 사용할 **Blob SAS 토큰**과 **Blob SAS URL**을 복사합니다. 해당 창을 닫으면 토큰 값에 다시 액세스할 수 없습니다.

이제 **employees.json** 파일에 액세스할 수 있는 보안 URL이 생겼으니 테스트해 보겠습니다.

1. 새 브라우저 탭을 열고 **Blob SAS URL**을 붙여넣습니다.
1. 브라우저에 표시되는 **employees.json** 파일의 내용이 다음과 같이 보여야 합니다.

    ```json
    {
        "employees": [
            {
                "EmployeeID": 1,
                "FirstName": "John",
                "LastName": "Doe",
                "Department": "HR"
            },
            {
                "EmployeeID": 2,
                "FirstName": "Jane",
                "LastName": "Smith",
                "Department": "Engineering"
            }
        ]
    }
    ```

### Blob Storage에서 Azure SQL 데이터베이스로 데이터 가져오기

이제 Azure Blob Storage에 호스팅된 **employees.json** 파일에서 Azure SQL 데이터베이스로 데이터를 가져올 준비가 되었습니다.

먼저 Azure SQL 데이터베이스에 **마스터 키**와 **데이터베이스 범위 자격 증명**을 만들어 시작해야 합니다.

1. SSMS **(SQL Server Management Studio)** 또는 **Azure Data Studio**를 사용하여 Azure SQL 데이터베이스에 연결합니다.
1. *아직 생성하지 않았다면* 다음 SQL 명령을 실행하여 마스터 키를 만듭니다.

    ```sql
    CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'YourStrongPassword!';
    ```

1. 그런 다음, 다음 SQL 명령을 실행하여 Azure Blob Storage에 액세스하기 위한 **데이터베이스 범위 자격 증명**을 만듭니다.

    ```sql
    CREATE DATABASE SCOPED CREDENTIAL MyBlobCredential
    WITH IDENTITY = 'SHARED ACCESS SIGNATURE', 
    SECRET = '<your-sas-token>';
    ```

    \<your-sas-token\>을 앞서 생성된 Blob SAS 토큰으로 바꿉니다.

1. 마지막으로 Azure Blob Storage에 액세스하려면 **데이터 원본**이 필요합니다. 다음 SQL 명령을 실행하여 **데이터 원본**을 만듭니다.

    ```sql
    CREATE EXTERNAL DATA SOURCE MyBlobDataSource
    WITH (
        TYPE = BLOB_STORAGE,
        LOCATION = 'https://<your-storage-account-name>.blob.core.windows.net',
        CREDENTIAL = MyBlobCredential
    );
    ```

    이름(\<your-storage-account-name\>)을 Azure Storage 계정 이름으로 바꿉니다.

이제 모든 것이 **employees.json** 파일에서 *Azure SQL 데이터베이스*로 데이터를 가져오도록 설정되었습니다.

다음 SQL 명령을 사용하여 *Azure Blob Storage*에 호스팅된 **employees.json** 파일에서 데이터를 가져옵니다.

```sql
SELECT EmployeeID
    , FirstName
    , LastName
    , Department
INTO dbo.employee_data
FROM OPENROWSET(
    BULK 'jsonfiles/employees.json',
    DATA_SOURCE = 'MyBlobDataSource',
    SINGLE_CLOB
) AS JSONData
CROSS APPLY OPENJSON(JSONData.BulkColumn, '$.employees')
WITH (
    EmployeeID INT '$.EmployeeID',
    FirstName NVARCHAR(50) '$.FirstName',
    LastName NVARCHAR(50) '$.LastName',
    Department NVARCHAR(50) '$.Department'
) AS EmployeeData;
```

이 명령은 *Azure Blob Storage*에 있는 **jsonfiles** 컨테이너에서 **employees.json** 파일을 읽고 Azure SQL 데이터베이스의 **employee_data** 테이블로 데이터를 가져옵니다.

이제 다음 SQL 명령을 실행하여 데이터 가져오기를 확인할 수 있습니다. 

```sql
SELECT * FROM dbo.employee_data;
```

**employees.json** 파일의 데이터를 **employee_data** 테이블로 가져온 것을 볼 수 있습니다.

---

## Azure 함수 앱을 사용하여 데이터 내보내기

이 랩의 이 부분에서는 Azure SQL 데이터베이스에서 데이터를 내보내는 C#으로 Azure 함수 앱을 만듭니다. 이 함수는 데이터를 검색하고 JSON 응답으로 반환합니다.

### Visual Studio Code에서 Azure 함수 앱 만들기

먼저 Visual Studio Code에서 Azure 함수 앱을 만들어 보겠습니다.

1. **Visual Studio Code**를 사용하여 ASP.NET 5 API 앱을 만드는 방법을 보여줍니다.
1. 탐색기 창에서 **/Allfiles/Labs/04/azure-functions** 폴더로 이동합니다.
1. **azure-functions** 폴더를 마우스 오른쪽 단추로 클릭하고 **통합 터미널에서 열기**를 선택합니다.
1. VS Code 터미널에서 다음 명령을 사용하여 Azure에 로그인합니다.

    ```bash
    az login
    ```

1. (선택) 여러 구독이 있는 경우 활성 구독을 설정합니다.

    ```bash
    az account set --subscription <your-subscription-id>
    ```

1. 다음 명령을 실행하여 Azure 함수 앱을 만듭니다.

    ```bash
    $functionappname = "YourUniqueFunctionAppName"
    $resourcegroup = "YourResourceGroupName"
    $location = "YourLocation"
    # NOTE - The following should be a new storage account name where your Azure function will reside.
    # It should not be the same Storage Account name used to store the JSON file
    $storageaccount = "YourStorageAccountName"

    az storage account create --name $storageaccount --location $location --resource-group $resourcegroup --sku Standard_LRS
    
    az functionapp create --resource-group $resourcegroup --consumption-plan-location $location --runtime dotnet --name  $functionappname --os-type Linux --storage-account $storageaccount --functions-version 4
    
    ```

    ***자리 표시자를 원하는 값으로 바꿉니다. json 파일에 사용된 스토리지 계정 이름을 사용하지 마세요. 이 스크립트에서는 Azure 함수 앱을 저장하기 위해 새 스토리지 계정을 만들어야 합니다.***


### Visual Studio Code에서 새 함수 앱 만들기

Visual Studio Code에서 새 함수를 만들어 Azure SQL 데이터베이스에서 데이터를 내보내겠습니다.

아직 추가하지 않았다면 Visual Studio Code에 Azure Functions 확장을 추가해야 할 수 있습니다. 확장 창에서 **Azure Functions**를 검색하여 설치하면 됩니다.

1. Visual Studio Code에서 **Ctrl+Shift+P**(Windows) 또는 **Cmd+Shift+P**(Mac)를 눌러 명령 팔레트를 엽니다.
1. **Azure Functions: 새 프로젝트 만들기**를 입력하고 선택합니다.
1. **함수 앱** 디렉터리를 선택합니다. GitHub 복제 리포지토리의 **/Allfiles/Labs/04/azure-functions** 폴더를 선택합니다.
1. 언어로 **C#** 을 선택합니다.
1. 런타임으로 **.Net 8.0 LTS**를 선택합니다.
1. 템플릿으로 **HTTP 트리거**를 선택합니다.
1. **ExportDataFunction** 함수를 호출합니다.
1. 네임스페이스 **Contoso.ExportFunction**를 만듭니다.
1. 함수에 **익명** 액세스 수준을 지정합니다.

### 데이터를 내보내기 위한 C# 코드 작성

1. Azure 함수 앱을 사용하려면 먼저 몇 가지 패키지를 설치해야 할 수 있습니다. 다음 명령을 실행하여 설치할 수 있습니다.

    ```bash
    dotnet add package Microsoft.Data.SqlClient
    dotnet add package Newtonsoft.Json
    dotnet restore
    
    npm install -g azure-functions-core-tools@4 --unsafe-perm true

    ```

1. 자리 표시자 함수 코드를 다음 C# 코드로 대체하여 Azure SQL 데이터베이스를 쿼리하고 결과를 JSON으로 반환합니다.

    ```csharp
    using System;
    using System.IO;
    using Microsoft.AspNetCore.Mvc;
    using Microsoft.Azure.WebJobs;
    using Microsoft.Azure.WebJobs.Extensions.Http;
    using Microsoft.AspNetCore.Http;
    using Microsoft.Extensions.Logging;
    using Microsoft.Data.SqlClient;
    using Newtonsoft.Json;
    using System.Collections.Generic;
    using System.Threading.Tasks;
    
    public static class ExportDataFunction
    {
        [FunctionName("ExportDataFunction")]
        public static async Task<IActionResult> Run(
            [HttpTrigger(AuthorizationLevel.Function, "get", Route = null)] HttpRequest req,
            ILogger log)
        {
            // Connection string to the database
            // NOTE: REPLACE THIS CONNECTION STRING WITH THE CONNECTION STRING OF YOUR AZURE SQL DATABASE
            string connectionString = "Server=tcp:yourserver.database.windows.net;Database=DataLabDatabase;User ID=youruserid;Password=yourpassword;Encrypt=True;";
            
            // List to hold employee data
            List<Employee> employees = new List<Employee>();
            
            try
            {
                // Establishing connection to the database
                using (SqlConnection conn = new SqlConnection(connectionString))
                {
                    await conn.OpenAsync();
                    var query = "SELECT EmployeeID, FirstName, LastName, Department FROM employee_data";
                    
                    // Executing the query
                    using (SqlCommand cmd = new SqlCommand(query, conn))
                    {
                        // Adding parameters to the query (if needed)
                        // cmd.Parameters.AddWithValue("@ParameterName", parameterValue);

                        using (SqlDataReader reader = await cmd.ExecuteReaderAsync())
                        {
                            // Reading data from the database
                            while (await reader.ReadAsync())
                            {
                                employees.Add(new Employee
                                {
                                    EmployeeID = (int)reader["EmployeeID"],
                                    FirstName = reader["FirstName"].ToString(),
                                    LastName = reader["LastName"].ToString(),
                                    Department = reader["Department"].ToString()
                                });
                            }
                        }
                    }
                }
            }
            catch (SqlException ex)
            {
                // Logging SQL errors
                log.LogError($"SQL Error: {ex.Message}");
                return new StatusCodeResult(500);
            }
            catch (System.Exception ex)
            {
                // Logging unexpected errors
                log.LogError($"Unexpected Error: {ex.Message}");
                return new StatusCodeResult(500);
            }
    
            // Returning the list of employees as a JSON response
            return new OkObjectResult(JsonConvert.SerializeObject(employees, Formatting.Indented));
        }
    
        // Employee class to hold employee data
        public class Employee
        {
            public int EmployeeID { get; set; }
            public string FirstName { get; set; }
            public string LastName { get; set; }
            public string Department { get; set; }
        }
    }
    ```

    ***conectionString**을 Azure SQL 데이터베이스에 대한 연결 문자열로 바꾸고 연결 문자열에 sqladmin 비밀번호도 입력해야 합니다.*

    > **참고:** 프로덕션 환경에서는 필요한 IP 주소로만 액세스를 제한합니다. 또한 SQL 인증 대신 데이터베이스에 액세스하기 위해 Azure 함수 앱에 관리 ID를 사용하는 것도 고려합니다. 자세한 내용은 [Azure SQL용 Microsoft Entra의 관리 ID](https://learn.microsoft.com/azure/azure-sql/database/authentication-azure-ad-user-assigned-managed-identity?azure-portal=true)를 참조하세요.

1. 함수 코드를 저장하고 **.csproj** 파일에 객체를 JSON으로 직렬화하기 위한 **Newtonsoft.Json** 패키지가 포함되어 있는지 확인합니다. 포함되지 않은 경우 추가합니다.

    ```xml
    <PackageReference Include="Newtonsoft.Json" Version="13.X.X" />
    ```

이제 Azure 함수 앱을 Azure에 배포할 차례입니다.

### Azure 함수 앱을 Azure에 배포

1. **Visual Studio Code** 통합 터미널에서 다음 명령을 실행하여 Azure 함수를 Azure에 배포합니다.

    ```bash
    func azure functionapp publish <your-function-app-name>
    ```

    ***<your-function-app-name>*** 을 Azure 함수 앱의 이름으로 바꿉니다.

1. 배포가 완료될 때가지 기다립니다.

### Azure 함수 앱 URL 가져오기

1. Azure Portal을 열고 Azure 함수 앱으로 이동합니다.
1. *개요* 섹션의 *함수* 탭에서 새 함수가 나열되면 해당 함수를 선택합니다.
1. **코드 + 테스트** 탭에서 **함수 URL 가져오기**를 선택합니다.
1. **기본값(함수 키)** 을 복사하면 곧 필요하게 될 것입니다. URL은 다음과 같이 표시되어야 합니다.
   
   ```url
   https://YourFunctionAppName.azurewebsites.net/api/ExportDataFunction?code=2pjO0HqRyz_13DHQg8ga-ysdDWbDU_eHdtlixbAHLVEGAzFuplomUg%3D%3D
   ```

### Azure 함수 앱 테스트

1. 배포가 완료되면 Visual Studio 코드 터미널에서 앞서 복사한 함수 키 URL로 HTTP 요청을 전송하여 함수를 테스트할 수 있습니다:

    ```bash
    curl https://<your-function-app-name>.azurewebsites.net/api/ExportDataFunction?code=<the function key embedded to your function URL>
    ```

1. 응답에는 ***employee_data*** 테이블에서 내보낸 데이터가 JSON 형식으로 포함되어야 합니다.

이 함수는 간단한 예시이지만 필터링, 정렬, 데이터 집계 등과 같은 더 복잡한 로직과 데이터 처리를 포함하도록 확장할 수 있습니다.  오류 처리, 로깅 및 보안 기능을 포함하도록 코드를 확장할 수도 있습니다.

### 리소스 정리

랩을 완료한 후 추가 비용이 발생하지 않도록 이 연습에서 만든 리소스를 삭제할 수 있습니다.

- Azure SQL 데이터베이스를 삭제합니다.
- Azure Storage 계정을 삭제합니다.
- Azure 함수 앱을 삭제합니다.
- 리소스가 포함된 리소스 그룹을 삭제합니다.

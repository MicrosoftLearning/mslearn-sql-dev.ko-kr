---
lab:
  title: Azure SQL Database용 데이터 API 개발
  module: Develop a Data API for Azure SQL Database
---

# Azure SQL Database용 데이터 API 개발

이 연습에서는 Azure Static Web Apps를 사용하여 Azure SQL 데이터베이스용 데이터 API를 개발하고 배포합니다. 이를 통해 데이터 API 작성기 구성을 설정하고 Azure Static Web App 환경 내에 배포하는 실습 경험을 제공합니다.

## 필수 조건

이 섹션을 시작하기 전에 다음을 확인하세요.

- 활성화된 Azure 구독
- Azure SQL Database, Azure Static Web Apps 및 GitHub에 대한 기본 지식
- 필요한 확장과 함께 설치된 Visual Studio Code.
- 리포지토리를 관리하기 위한 GitHub 계정.

## 환경 설정

이 연습에 대한 환경을 설정하기 위해 수행해야 하는 몇 가지 단계가 있습니다.

### Visual Studio Code 확장을 설치합니다.

연습을 시작하기 전에 Visual Studio Code 확장을 설치해야 합니다.

1. Visual Studio Code를 엽니다.
1. Visual Studio Code에서 터미널 창을 엽니다.
1. 다음 명령을 사용하여 Static Web Apps CLI를 설치합니다.

    ```bash
    npm install -g @azure/static-web-apps-cli
    ```

1. 다음 명령을 실행하여 데이터 API 작성기 CLI를 설치합니다.

    ```bash
    dotnet tool install --global Microsoft.DataApiBuilder
    ```

이제 Visual Studio Code가 필요한 확장으로 설정됩니다.

### Azure SQL Database 만들기

아직 수행하지 않은 경우 Azure SQL Database를 만들어야 합니다.

1. [Azure Portal](https://portal.azure.com?azure-portal=true)에 로그인합니다. 
1. **Azure SQL** 페이지로 이동한 다음 **+ 만들기**를 선택합니다.
1. **SQL Database**, *단일 데이터베이스*를 선택하고 **만들기** 단추를 클릭합니다.
1. **SQL 데이터베이스 만들기** 대화 상자에서 필요한 정보를 입력하고 **확인**을 선택합니다(다른 모든 옵션은 기본값으로 둡니다).

    | 설정 | 값 |
    | --- | --- |
    | 무료 서버리스 제품 | *제품 적용* |
    | 구독 | 구독 |
    | 리소스 그룹 | *리소스 그룹 선택 또는 새로 만들기* |
    | 데이터베이스 이름 | *MyDB* |
    | 서버 | ***새로 만들기** 링크를 선택합니다* |
    | 서버 이름 | *고유한 이름 선택* |
    | 위치 | *위치를 선택합니다* |
    | 인증 방법 | *SQL 인증 사용* |
    | 서버 관리자 로그인 | *sqladmin* |
    | 암호 | *암호를 입력합니다.* |
    | 암호 확인 | *비밀번호를 확인합니다* |

1. **검토 + 만들기**, **만들기**를 차례로 선택합니다.
1. 배포가 완료되면 만든 Azure SQL 데이터베이스 *서버*로 이동합니다.
1. 왼쪽 창의 **보안**에서 **네트워킹**을 선택합니다. 방화벽 규칙에 IP 주소를 추가합니다.
1. **저장**을 선택합니다.

### 데이터베이스에 샘플 데이터 추가

이제 Azure SQL Database가 있으므로 몇 가지 샘플 데이터를 추가해야 합니다. 이렇게 하면 API가 실행되고 나서 테스트하는 데 도움이 됩니다.

1. 새로 만든 Azure SQL Database로 이동합니다.
1. Azure Portal의 **쿼리 편집기**를 사용하여 다음 SQL 스크립트를 실행합니다.

    ```sql
    CREATE TABLE [dbo].[Employees]
    (
        EmployeeID INT PRIMARY KEY,
        FirstName NVARCHAR(50),
        LastName NVARCHAR(50),
        Department NVARCHAR(50)
    );
    
    INSERT INTO [dbo].[Employees] VALUES (1,'John', 'Smith', 'Accounting');
    INSERT INTO [dbo].[Employees] VALUES (2,'Michelle', 'Sanchez', 'IT');
    
    SELECT * FROM [dbo].[Employees];
    ```

### GitHub에서 기본 웹앱 만들기

Azure Static Web App을 만들려면 GitHub에서 기본 웹앱을 만들어야 합니다.

1. GitHub에서 기본 웹앱을 만들려면 [바닐라 웹 사이트 생성](https://github.com/staticwebdev/vanilla-basic/generate)으로 이동합니다.
1. 리포지토리 템플릿이 **staticwebdev/vanilla-basic**으로 설정되어 있는지 확인합니다.
1. ***소유자***에서 계정을 선택합니다.
1. ***리포지토리 이름***에 **my-sql-repo**라는 이름을 입력합니다.
1. 리포지토리를 **비공개**로 설정합니다.
1. **리포지토리 만들기** 버튼을 선택합니다.

## Azure Static Web App 만들기

먼저 정적 웹앱을 만든 다음 데이터 API 작성기 구성을 추가하겠습니다.

1. Azure Portal에서 **Static Web Apps** 페이지로 이동합니다.
1. **+ 만들기**를 선택합니다.
1. **Static Web App 만들기** 대화 상자에서 다음 정보를 입력합니다(다른 모든 옵션은 기본값으로 둡니다).

    | 설정 | 값 |
    | --- | --- |
    | 구독 | 구독 |
    | 리소스 그룹 | *리소스 그룹 선택 또는 새로 만들기* |
    | 속성 | *고유 이름* |
    | 호스팅 계획 원본 | *GitHub* |
    | GitHub 계정 | *계정 선택* |
    | 조직 | *GitHub 사용자 이름일 가능성이 높습니다.* |
    | 리포지토리 | *이전 단계에서 만든 리포지토리를 선택합니다*. |
    | 지점 | *main* |

1. **검토 + 만들기**를 선택한 다음, **만들기**를 선택합니다.
1. 배포되면 리소스로 이동합니다.
1. **브라우저에서 앱 보기** 버튼을 선택하면 환영 메시지와 함께 간단한 웹 페이지가 표시됩니다. 이 탭을 닫을 수 있습니다.

## 데이터 API 작성기 구성 파일 만들기

이제 데이터 API 작성기 구성을 Azure Static Web App에 추가할 차례입니다. 데이터 API 작성기 구성을 추가하려면 GitHub 리포지토리에 새 파일을 만들어야 합니다.

1. Visual Studio Code에서 앞서 만든 GitHub 리포지토리를 복제합니다.
1. Visual Studio Code에서 터미널 창을 엽니다.
1. 다음 명령을 실행하여 새 데이터 API 작성기 구성 파일을 만듭니다.

    ```bash
    swa db init --database-type "mssql"
    ```

    그러면 *swa-db-connections*라는 새 폴더가 생성되고 해당 폴더 안에 *staticwebapp.database.config.json*이라는 파일이 만들어집니다.

1. 다음 명령을 실행하여 구성 파일에 데이터베이스 엔터티를 추가할 수 있습니다.

    ```bash
    dab add "Employees" --source "dbo.Employees" --permissions "anonymous:*" --config "swa-db-connections/staticwebapp.database.config.json"
    ```

1. *staticwebapp.database.config.json* 파일의 내용을 검토합니다. 
1. 변경 내용을 Git 리포지토리에 커밋하고 푸시합니다.

## 데이터베이스 연결 구성

1. Azure 포털에서 만든 Azure Static Web App으로 이동합니다.
1. **설정** 섹션에서 **데이터베이스 연결**을 선택합니다.
1. **기존 데이터베이스 연결**을 선택합니다.
1. **데이터베이스 연결* 대화 상자에서 다음과 같은 추가 설정을 사용하여 이전에 만든 Azure SQL 데이터베이스를 선택합니다.

    | 설정 | 값 |
    | --- | --- |
    | 데이터베이스 유형 | *Azure SQL Database* |
    | 인증 유형 | *연결 문자열* |
    | 사용자 이름 | *관리 사용자 이름* |
    | 암호 | *관리 사용자에게 제공한 비밀번호* |
    | 확인 체크박스 | *선택* |

   > **참고:** 프로덕션 환경에서는 필요한 IP 주소로만 액세스를 제한합니다. 또한 Static Web App에 관리 ID를 사용하여 SQL 인증 대신 데이터베이스에 액세스하는 것도 고려합니다. 자세한 내용은 [Azure SQL용 Microsoft Entra의 관리 ID](https://learn.microsoft.com/azure/azure-sql/database/authentication-azure-ad-user-assigned-managed-identity?azure-portal=true)를 참조하세요.
1. **링크**를 선택합니다.

## 데이터 API 엔드포인트 테스트

이제 데이터 API 엔드포인트만 테스트해야 합니다.

1. Azure 포털에서 만든 Azure Static Web App으로 이동합니다.
1. 개요 페이지에서 웹앱의 URL을 복사합니다.
1. 새 브라우저 탭을 열고 URL을 붙여넣습니다. **Vanilla JavaScript App**이라는 메시지가 표시된 간단한 웹 페이지가 계속 표시됩니다.
1. URL 끝에 **/data-api**를 추가하고 **Enter** 키를 누릅니다. 데이터 API가 작동 중임을 나타내는 **Healthy**가 표시되어야 합니다.
1. URL 끝에 **/data-api/rest/Employees**를 추가하고 **Enter** 키를 누릅니다. 이전에 Azure SQL Database에 추가한 샘플 데이터가 표시됩니다.

Azure Static Web Apps를 사용하여 Azure SQL Database용 데이터 API를 성공적으로 개발하고 배포했습니다.

## 정리

본인 소유의 구독으로 이 모듈을 진행하고 있는 경우에는 프로젝트가 끝날 때 여기에서 만든 리소스가 계속 필요한지 확인하는 것이 좋습니다. 

리소스를 불필요하게 실행하면 추가 비용이 발생할 수 있습니다. 리소스를 개별적으로 삭제하거나 [Azure Portal](https://portal.azure.com?azure-portal=true)에서 전체 리소스 집합을 삭제할 수 있습니다.

## 자세한 정보

Azure SQL 데이터베이스용 데이터 API 작성기에 대한 자세한 내용은 [Azure 데이터베이스용 데이터 API 작성기란?](https://learn.microsoft.com/azure/data-api-builder/overview?azure-portal=true)을 참조합니다.

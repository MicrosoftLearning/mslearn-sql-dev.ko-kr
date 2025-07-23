---
lab:
  title: Azure SQL 데이터베이스 프로젝트를 위한 CI/CD 파이프라인 구성 및 배포
  module: Develop for an Azure SQL Database
---

# Azure SQL 데이터베이스 프로젝트를 위한 CI/CD 파이프라인 구성 및 배포

이 연습에서는 Visual Studio Code와 GitHub Actions를 사용하여 Azure SQL Database 프로젝트에 대한 CI/CD 파이프라인을 만들고, 구성하고, 배포합니다. 이를 통해 Azure SQL Database 프로젝트에 대한 CI/CD 파이프라인을 설정하는 프로세스에 익숙해질 수 있습니다.

이 연습을 완료하는 데 약 **30**분 정도 소요됩니다.

## 시작하기 전에

이 연습을 시작하기 전에 다음을 수행해야 합니다.

- 리소스를 만들고 관리할 수 있는 적절한 권한이 있는 Azure 구독.
- [Visual Studio Code](https://code.visualstudio.com/download)가 다음 확장과 함께 컴퓨터에 설치되어 있습니다.
  - [SQL Database 프로젝트](https://marketplace.visualstudio.com/items?itemName=ms-mssql.mssql).
  - [GitHub 끌어오기 요청](https://marketplace.visualstudio.com/items?itemName=GitHub.vscode-pull-request-github).
- GitHub 계정.
- *GitHub Actions* 파이프라인에 대한 기본 지식.

## Azure SQL Database 만들기

먼저 새 Azure SQL 데이터베이스를 만들어야 합니다.

1. [Azure Portal](https://portal.azure.com?azure-portal=true)에 로그인합니다. 
1. **Azure SQL** 페이지로 이동한 다음 **+ 만들기**를 선택합니다.
1. **SQL Database**, *단일 데이터베이스*를 선택하고 **만들기** 단추를 클릭합니다.
1. **SQL 데이터베이스 만들기** 대화 상자에서 필요한 정보를 입력하고 **확인**을 선택한 후 다른 모든 옵션은 기본 설정으로 둡니다.

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
1. **Azure 서비스 및 리소스에서 이 서버에 액세스할 수 있도록 허용** 옵션을 선택합니다. 이 옵션을 사용하면 GitHub Actions에서 데이터베이스에 액세스할 수 있습니다.

    > **참고:** 프로덕션 환경에서는 필요한 IP 주소로만 액세스를 제한합니다. 또한 SQL 인증 대신 데이터베이스에 액세스하려면 GitHub Action에 관리되는 ID를 사용하는 것도 고려합니다. 자세한 내용은 [Azure SQL용 Microsoft Entra의 관리 ID](https://learn.microsoft.com/azure/azure-sql/database/authentication-azure-ad-user-assigned-managed-identity?azure-portal=true)를 참조하세요.

1. **저장**을 선택합니다.

## GitHub 리포지토리 설정하기

다음으로 새 GitHub 리포지토리를 설정해야 합니다.

1. [GitHub](https://github.com) 웹 사이트를 엽니다.
1. GitHub 계정에 로그인합니다.
1. 계정 아래의 **리포지토리**로 이동하여 **새로 만들기**를 선택합니다.
1. **소유자**에 대한 계정을 선택합니다. **my-sql-db-repo**라는 이름을 입력합니다.
1. 리포지토리를 **비공개**로 설정합니다.
1. **리포지토리 만들기**를 선택합니다.

### Visual Studio Code 확장을 설치하고 리포지토리를 복제합니다.

리포지토리를 복제하기 전에 필요한 **Visual Studio Code** 확장을 설치했는지 확인합니다. 지침은 **시작하기 전에** 섹션을 참조하세요.

1. Visual Studio Code에서 **보기** > **명령 팔레트**를 선택합니다.
1. 명령 팔레트에서 `Git: Clone`을(를) 입력하고 선택합니다.
1. 이전 단계에서 생성한 리포지토리의 URL을 입력하고 **복제**를 선택합니다. URL은 다음 형식(*https://github.com/<your_account>/<your_repository>.git*)을 따라야 합니다.
1. 리포지토리 파일을 저장할 폴더를 선택하거나 만듭니다.

## Azure SQL 데이터베이스 프로젝트 만들기 및 구성

Visual Studio의 Azure SQL 데이터베이스 프로젝트를 사용하면 데이터베이스 스키마 및 데이터를 개발, 빌드, 테스트 및 게시할 수 있습니다. 이 섹션에서는 프로젝트를 만들고 이전에 설정한 Azure SQL Database에 연결하도록 구성합니다.

1. Visual Studio Code에서 **보기** > **명령 팔레트**를 선택합니다.
1. 명령 팔레트에서 `Database projects: New`을(를) 입력하고 선택합니다.
    > **참고:** mssql 확장용 SQL 도구 서비스를 설치하는 데 몇 분 정도 걸릴 수 있습니다.
1. **Azure SQL 데이터베이스**를 선택합니다.
1. **MyDBProj**라는 이름을 입력하고 **Enter** 키를 눌러 확인합니다.
1. 복제된 GitHub 리포지토리 폴더를 선택하여 프로젝트를 저장합니다.
1. **SDK 스타일 프로젝트**의 경우 **예(권장)** 를 선택합니다.
    > **참고:** 새 프로젝트가 **MyDBProj**라는 이름으로 만들어집니다.

### 프로젝트에 새 SQL 파일을 만듭니다.

Azure SQL 데이터베이스 프로젝트가 생성되었으면 프로젝트에 새 SQL 파일을 추가하여 새 테이블을 만들어 보겠습니다.

1. Visual Studio Code에서 왼쪽의 작업 표시줄에 있는 **데이터베이스 프로젝트** 아이콘을 선택합니다.
1. 프로젝트 이름을 마우스 오른쪽 단추로 클릭하고 **테이블 추가**를 선택합니다.
1. 테이블 이름을 **Employees**로 지정하고 **Enter** 키를 누릅니다.
1. 기존 스크립트를 다음 코드로 바꿉니다.

    ```sql
    CREATE TABLE [dbo].[Employees]
    (
        EmployeeID INT PRIMARY KEY,
        FirstName NVARCHAR(50),
        LastName NVARCHAR(50),
        Department NVARCHAR(50)
    );
    ```

1. 편집기를 닫습니다. 프로젝트에 `Employees.sql` 파일이 저장되어 있는 것을 확인합니다.

## 변경 내용을 리포지토리에 커밋합니다.

Azure SQL 데이터베이스 프로젝트가 생성되고 테이블 스크립트가 프로젝트에 추가되었으므로 이제 변경 사항을 리포지토리에 커밋해 보겠습니다.

1. Visual Studio Code에서 왼쪽의 작업 표시줄에 있는 **소스 제어** 아이콘을 선택합니다.
1. *Created project and added a create table script*라는 메시지를 입력합니다.
1. **커밋**을 선택하여 변경 내용을 커밋합니다.
1. 줄임표 아래에서 **푸시**를 선택하여 변경 내용을 리포지토리에 푸시합니다.

## 리포지토리에서 변경 사항을 확인합니다.

변경 내용을 푸시했으므로 GitHub 리포지토리에서 확인해 보겠습니다.

1. [GitHub](https://github.com) 웹 사이트를 엽니다.
1. **my-sql-db-repo** 리포지토리로 이동합니다.
1. **<> 코드** 탭에서 **MyDBProj** 폴더를 엽니다.
1. **Employees.sql** 파일의 변경 사항이 최신 상태인지 확인합니다.

## GitHub Actions를 사용하여 CI(연속 통합)를 설정합니다.

GitHub Actions를 사용하면 GitHub 리포지토리에서 직접 소프트웨어 개발 워크플로를 자동화, 사용자 지정 및 실행할 수 있습니다. 이 섹션에서는 데이터베이스에 새 테이블을 만들어 Azure SQL 데이터베이스 프로젝트를 빌드하고 테스트하는 GitHub Actions 워크플로를 구성합니다.

### 서비스 주체 만들기

1. Azure Portal 오른쪽 상단에 있는 **Cloud Shell** 아이콘을 선택합니다. `>_` 기호처럼 보입니다. 메시지가 표시되면 셸 유형으로 **Bash**를 선택합니다.

1. Cloud Shell 터미널에서 다음 명령을 실행합니다. `<your_subscription_id>` 및 `<your_resource_group_name>` 값을 실제 값으로 바꿉니다. 이 값은 Azure Portal의 **구독** 및 **리소스 그룹** 페이지에서 확인할 수 있습니다.

    ```azurecli
    az ad sp create-for-rbac --name "MyDBProj" --role contributor --scopes /subscriptions/<your_subscription_id>/resourceGroups/<your_resource_group_name>
    ```

    텍스트 편집기를 열고 이전 명령의 출력을 사용하여 다음과 유사한 자격 증명 조각을 만듭니다.
    
    ```
    {
    "clientId": <your_service_principal_appId>,
    "clientSecret": <your_service_principal_password>,
    "tenantId": <your_service_principal_tenant>,
    "subscriptionId": <your_subscription_id>
    }
    ```

1. 텍스트 편집기를 열어 둡니다. 다음 섹션에서 이를 참조하겠습니다.

### 리포지토리에 비밀을 추가합니다.

1. GitHub 리포지토리에서 **설정**을 선택합니다.
1. **비밀 및 변수**, **작업**을 차례로 선택합니다.
1. **비밀** 탭에서 **새 리포지토리 비밀**을 선택하고 다음 정보를 제공합니다.

    | 속성 | 값 |
    | --- | --- |
    | AZURE_CREDENTIALS | 이전 섹션에서 복사한 서비스 주체 출력.|
    | AZURE_CONN_STRING | 연결 문자열 |
   
    연결 문자열은 다음과 유사할 수 있습니다.

    ```Server=<your_sqldb_server>.database.windows.net;Initial Catalog=MyDB;Persist Security Info=False;User ID=sqladmin;Password=<your_password>;Encrypt=True;Connection Timeout=30;```

### GitHub Actions 워크플로 만들기

1. GitHub 리포지토리에서 **작업** 탭을 선택합니다.
1. **직접 워크플로 설정** 링크를 선택합니다.
1. 아래 코드를 **main.yml** 파일에 복사합니다. 이 코드에는 데이터베이스 프로젝트를 빌드하고 배포하는 단계가 포함되어 있습니다.

    {% raw %}
    ```yaml
    name: Build and Deploy SQL Database Project
    on:
      push:
        branches:
          - main
    jobs:
      build:
        permissions:
          contents: 'read'
          id-token: 'write'
          
        runs-on: ubuntu-latest  # Can also use windows-latest depending on your environment
        steps:
          - name: Checkout repository
            uses: actions/checkout@v3

        # Install the SQLpackage tool
          - name: sqlpack install
            run: dotnet tool install -g microsoft.sqlpackage
    
          - name: Login to Azure
            uses: azure/login@v1
            with:
              creds: ${{ secrets.AZURE_CREDENTIALS }}
    
          # Build and Deploy SQL Project
          - name: Build and Deploy SQL Project
            uses: azure/sql-action@v2.3
            with:
              connection-string: ${{ secrets.AZURE_CONN_STRING }}
              path: './MyDBProj/MyDBProj.sqlproj'
              action: 'publish'
              build-arguments: '-c Release'
              arguments: '/p:DropObjectsNotInSource=true'  # Optional: Customize as needed
    ```
    {% endraw %}
   
      YAML 파일의 **SQL 프로젝트 빌드 및 배포** 단계는 `AZURE_CONN_STRING` 비밀에 저장된 연결 문자열을 사용하여 Azure SQL 데이터베이스에 연결합니다. 이 작업은 SQL 프로젝트 파일의 경로를 지정하고, 프로젝트를 배포하기 위해 게시할 작업을 설정하며, 릴리스 모드에서 컴파일할 빌드 인수를 포함합니다. 또한 `/p:DropObjectsNotInSource=true` 인수를 사용하여 배포 중에 원본에 없는 모든 개체가 대상 데이터베이스에서 삭제되도록 합니다.

1. 변경 내용을 커밋합니다.

### GitHub Actions 워크플로 테스트

1. GitHub 리포지토리에서 **작업** 탭을 선택합니다.
1. **SQL 데이터베이스 프로젝트 빌드 및 배포** 워크플로를 선택합니다.
    > **참고:** 워크플로가 진행 중인 것을 볼 수 있습니다. 완료할 때까지 기다립니다. 이미 완료된 경우 최신 실행을 선택하여 세부 정보를 확인합니다.

### Azure SQL 데이터베이스의 변경 사항을 확인합니다.

Azure SQL 데이터베이스 프로젝트를 빌드하고 배포하기 위해 GitHub Actions 워크플로를 설정했으면 이제 Azure SQL 데이터베이스의 변경 사항을 확인할 차례입니다.

1. [Azure Portal](https://portal.azure.com?azure-portal=true)에 로그인합니다. 
1. **MyDB** SQL 데이터베이스로 이동합니다.
1. **쿼리 편집기**를 선택합니다.
1. **sqladmin** 자격 증명을 사용하여 데이터베이스에 연결합니다.
1. **테이블** 섹션에서 **Employees** 테이블이 생성되었는지 확인합니다. 필요한 경우 새로고침합니다.

Azure SQL 데이터베이스 프로젝트를 빌드하고 배포하기 위한 GitHub Actions 워크플로를 성공적으로 설정했습니다.

## 정리

본인 소유의 구독으로 이 모듈을 진행하고 있는 경우에는 프로젝트가 끝날 때 여기서 만든 리소스가 계속 필요한지 확인하는 것이 좋습니다. 

리소스를 불필요하게 실행하면 추가 비용이 발생할 수 있습니다. 리소스를 개별적으로 삭제하거나 [Azure Portal](https://portal.azure.com?azure-portal=true)에서 전체 리소스 집합을 삭제할 수 있습니다.

## 자세한 정보

Azure SQL 데이터베이스용 SQL 데이터베이스 프로젝트 확장에 대한 자세한 내용은 [SQL 데이터베이스 프로젝트 확장으로 시작](https://learn.microsoft.com/azure-data-studio/extensions/sql-database-project-extension-getting-started?azure-portal=true)을 참조하세요.

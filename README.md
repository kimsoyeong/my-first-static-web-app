
# React basic ([참고 URL](https://learn.microsoft.com/ko-kr/azure/app-service/quickstart-nodejs?tabs=linux&pivots=development-environment-azure-portal))

[Azure Static Web Apps](https://docs.microsoft.com/azure/static-web-apps/overview) allows you to easily build [React](https://reactjs.org/) apps in minutes. Use this repo with the [React quickstart](https://docs.microsoft.com/azure/static-web-apps/getting-started?tabs=react) to build and customize a new static site.

This project was bootstrapped with [Create React App](https://github.com/facebook/create-react-app).

<br />

**정적 파일 배포 (Azure Static Web Apps 또는 Web App):**

- React의 build 디렉토리를 Azure에 업로드.
- 서버 측 설정 없이 간단한 배포.

**Node.js를 사용한 배포 (Azure Web App):**

- serve와 같은 도구를 사용해 정적 파일을 서빙.
- Node.js 애플리케이션처럼 동작.


<br />

## Azure Web App의 동작 방식
1. 정적 파일 배포:
   - build 디렉토리에 있는 정적 파일을 자동으로 서빙하지 않음.
   - 따라서, 정적 파일을 서빙하려면 `serve -s build`와 같은 명령이 필요.
2. Node.js 애플리케이션:
   - Azure Web App (AWA)은 Node.js 프로젝트에서 기본적으로 **`package.json`의 `start` 스크립트를 기준으로 애플리케이션을 실행**. (다시 말해, AWA는 `npm start`에 의해 실행되는 작업을 수행한다.)
   - React 애플리케이션은 기본적으로 정적 파일을 서빙하지 않으므로, `serve`와 같은 도구의 명시적 설치 필요.

> React 프로젝트는 기본적으로 3000 포트를 통해 서빙된다.
> 
> 하지만, Azure Web App은 80(HTTP) 또는 443(HTTPS) 포트를 통해 애플리케이션에 접속하게 한다.
> 
> `serve` 패키지가 자동적으로 process.env.PORT 환경 변수를 인식하기 때문에, 추가 설정 없이도 Azure Web App에서 정상 동작한다.

<br />

## GitHub Actions를 활용한 React.js 프로젝트의 Azure Web App 배포 방법

1. Azure Web App 생성

2. React 프로젝트 설정
   - 터미널에서 `npm install serve --save` 실행
   - `package.json` 수정
     ```json
     "scripts": {
       "start": "serve -s build",  # 정적 파일 서빙
       "build": "react-scripts build"
       ...
     }
     ```

3. GitHub Actions 워크플로 파일: (.github/workflows/azure-webapps-react.yml)

  ```yml
  # 워크플로 이름
  name: Deploy React App to Azure Web App
  
  on:
    push:
      branches: [ "main" ]  # main 브랜치에 푸시될 때 실행
    workflow_dispatch:       # 수동으로 실행 가능
  
  env:
    AZURE_WEBAPP_NAME: your-app-name    # Azure Web App 이름 (반드시 수정)
    NODE_VERSION: '20.x'                # Node.js 버전
  
  permissions:
    contents: read
  
  jobs:
    build:
      runs-on: ubuntu-latest
      steps:
        # 1. 레포지토리 체크아웃
        - uses: actions/checkout@v4
  
        # 2. Node.js 설정
        - name: Set up Node.js
          uses: actions/setup-node@v4
          with:
            node-version: ${{ env.NODE_VERSION }}
            cache: 'npm'
  
        # 3. 의존성 설치 및 빌드: 정적 빌드 생성
        - name: Install dependencies and build
          run: |
            npm install
            npm run build
  
        # 4. 빌드 결과 업로드 (배포용 아티팩트 생성)
        - name: Upload build artifact
          uses: actions/upload-artifact@v3
          with:
            name: build
            path: build  # React.js의 기본 빌드 디렉토리
  
    deploy:
      permissions:
        contents: none
      runs-on: ubuntu-latest
      needs: build
      steps:
        # 5. 빌드 아티팩트 다운로드
        - name: Download build artifact
          uses: actions/download-artifact@v3
          with:
            name: build
  
        # 6. Azure Web App에 배포
        - name: Deploy to Azure Web App
          id: deploy
          uses: azure/webapps-deploy@v2
          with:
            app-name: ${{ env.AZURE_WEBAPP_NAME }}
            publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
            package: build  # 빌드된 정적 파일이 있는 디렉토리
  ```

3. 비밀 변수 설정:

  - AZURE_WEBAPP_PUBLISH_PROFILE를 GitHub 저장소의 비밀 변수로 설정 필요. Azure 포털에서 앱의 “게시 프로필”을 다운로드하여 구성.

4. 테스트 배포
   - main 브랜치에 코드 푸시 또는 GitHub Actions에서 수동으로 워크플로 실행.

<br />

#### Azure Web App의 시작 명령 수동 설정 (선택 사항)
만약 Azure Web App에서 자동으로 `serve -s build`를 실행하지 않는 경우, 아래와 같이 시작 명령을 수동으로 설정 가능하다.
1. Azure 포털에서 설정:
   - **Azure Web App -> 구성 -> 일반 설정 -> 시작 명령** 필드에 아래 명령 추가
     ```bash
     npx serve -s build
     ```
     
2. Azure CLI를 사용한 설정:
   CLI를 사용해 Azure Web App의 시작 명령을 설정하려면 다음을 실행:
   ```bash
   az webapp config set --resource-group <리소스 그룹명> --name <웹앱 명> --startup-file "npx serve -s build"
   ```

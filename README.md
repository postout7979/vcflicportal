# VCF-Lic-MGMT (배포 패키지)

VMware Cloud Foundation(VCF) 환경의 라이센스를 관리하기 위한 사내 도구입니다. 공급자(벤더) 라이센스 데이터, Broadcom CSP 데이터, 그리고 실제 운영 중인 vCenter/VCF Operations에서 수집한 실사용 현황을 "프로젝트" 단위로 모아 비교·관리합니다.

- **백엔드**: Python/FastAPI + PostgreSQL
- **프론트엔드**: React + Vite (TypeScript)
- **실행 방식**: Docker Compose (아래 이미지들을 그대로 불러와 실행)

이 폴더에는 소스 코드 대신 **빌드가 끝난 Docker 이미지**가 들어 있습니다. 별도의 빌드 과정 없이 이미지를 불러오기만 하면 바로 실행할 수 있습니다.

## 폴더 구성

```
vcf-ops-release/
├── docker-compose.yml       실행용 Compose 파일 (이미지 참조)
├── README.md                이 파일
└── images/
    ├── vcf-ops-backend.tar.gz
    ├── vcf-ops-frontend.tar.gz
    └── postgres-16.tar.gz
```

## 실행 방법

1. 이 폴더(`vcf-ops-release/`) 전체를 대상 호스트로 복사합니다 (`scp`, USB 등).
2. 대상 호스트에 Docker / Docker Compose가 설치되어 있어야 합니다.
3. 이미지를 불러옵니다.

   ```bash
   cd vcf-ops-release
   docker load -i images/vcf-ops-backend.tar.gz
   docker load -i images/vcf-ops-frontend.tar.gz
   docker load -i images/postgres-16.tar.gz
   ```

4. 애플리케이션을 실행합니다.

   ```bash
   docker compose up -d
   ```

5. 브라우저에서 접속합니다.

   - 화면: `https://<호스트IP>:5173` (자체 서명 인증서라 브라우저 보안 경고가 뜨는데, "고급 > 계속 진행"으로 넘어가면 됩니다)
   - API 문서(참고용): `http://<호스트IP>:8000/docs`
   - 기본 관리자 계정: `admin` / `admin` (최초 접속 후 반드시 비밀번호를 변경하세요)

중지/삭제: `docker compose down` (데이터까지 지우려면 `docker compose down -v`)

## docker public repository로부터 이미지 가져오기 및 실행
- DB는 공식 docker hub에서 가져오기
- frontend, backend는 postout7979에 업로드된 이미지를 repository에서 가져오기

1. docker-compose.yml 파일의 이미지 경로를 수정

    ```bash
    services:
      db:
        image: postgres:16
        restart: unless-stopped
        environment:
          POSTGRES_USER: vcf
          POSTGRES_PASSWORD: vcf
          POSTGRES_DB: vcf_lic_mgmt
        ports:
          - "5432:5432"
        volumes:
          - db_data:/var/lib/postgresql/data
        healthcheck:
          test: ["CMD-SHELL", "pg_isready -U vcf -d vcf_lic_mgmt"]
          interval: 5s
          timeout: 5s
          retries: 10
    
      backend:
        image: postout7979/vcf-ops-backend:latest
        restart: unless-stopped
        environment:
          DATABASE_URL: postgresql://vcf:vcf@db:5432/vcf_lic_mgmt
        ports:
          - "8000:8000"
        depends_on:
          db:
            condition: service_healthy
    
      frontend:
        image: postout7979/vcf-ops-frontend:latest
        restart: unless-stopped
        environment:
          API_PROXY_TARGET: http://backend:8000
        ports:
          - "5173:5173"
        depends_on:
          - backend
    
    volumes:
      db_data:
    ```

2. 애플리케이션을 실행합니다.

   ```bash
   docker compose up -d
   ```

## 데이터에 대한 중요 안내

이 패키지에는 **데이터베이스 내용은 포함되어 있지 않습니다** (이미지는 코드만 담고 있고, 실제 데이터는 Docker volume `db_data`에 별도로 저장됩니다). 처음 `docker compose up`을 실행하면 **완전히 빈 상태**(관리자 계정만 있는 초기 상태)로 시작됩니다.

기존에 쓰던 다른 환경의 데이터(프로젝트, 업로드한 라이센스 데이터, 사용자 계정, vCenter/VCF 연결 정보 등)를 그대로 옮기고 싶다면:

1. 기존 환경의 **설정 > 백업** 탭에서 "백업 다운로드"로 `.sql` 파일을 받습니다.
2. 이 패키지로 새로 띄운 환경에 로그인한 뒤, 같은 **설정 > 백업** 탭에서 그 `.sql` 파일을 업로드해 복구합니다.

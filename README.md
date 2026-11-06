# LoanOps

**금융 공공 API에서 수집한 대출 데이터를 검증한 뒤, 현재 조회 가능한 데이터로 게시하고 이력을 확인하는 운영 시스템입니다.** Spring Boot와 PostgreSQL로 수집·정규화·품질 검증 파이프라인을 만들고, React 화면에서 수집 상태와 게시 결과를 조회할 수 있게 했습니다.

## 왜 만들었나

금융감독원 금융상품 정보와 서울시 정책연계 대출 집계는 응답 형식과 데이터 의미가 다릅니다. API 호출이 성공해도 원천 데이터가 누락되거나 값이 잘못되면 화면에 이전의 정상 데이터를 계속 보여줘야 합니다. 이 문제를 다루기 위해 원본, 정규화 결과, 검증 결과, 현재 게시 데이터를 별도로 관리했습니다.

## 담당한 구현

- 금융감독원 FinLife JSON과 서울시 XML 응답을 수집하고, 실행 상태와 원본 본문·해시를 기록했습니다. 서울시 다중 페이지 응답은 하나의 실행 이력에 묶어 전체 페이지가 모였을 때만 다음 단계로 넘깁니다.
- 상품·상품 조건과 서울시 집계 통계를 각각 정규화했습니다. 기관명은 확인된 코드와 명시적 매핑으로만 연결하고, 판단할 수 없는 값은 미매핑 상태로 남겼습니다.
- 필수값, 날짜, 금리·건수 범위를 검사하고 오류 근거와 격리 이력을 저장했습니다. 검증에 실패한 실행은 게시하지 않아 마지막 정상 데이터를 유지합니다.
- 게시 대상이 이전 값과 같으면 변경으로 기록하지 않고, 실제 값이 달라졌을 때만 현재 조회 포인터와 변경 이력을 함께 갱신했습니다.
- 운영 조회 API와 React 화면을 구현해 수집 실행, 품질 오류, 게시 결과, 현재 상품·통계를 확인할 수 있게 했습니다.

금융감독원 상품 조건과 서울시 대출 집계는 서로 다른 데이터입니다. 두 데이터를 같은 상품의 금리처럼 합치지 않습니다.

## 코드 구성

```text
src/main/java/com/loanops/source/         API 수집과 실행 이력
src/main/java/com/loanops/normalization/  원천별 정규화
src/main/java/com/loanops/publication/    품질 검증·격리·게시
src/main/java/com/loanops/api/            운영 조회 API
src/main/resources/db/migration/         데이터베이스 스키마
src/test/                                핵심 동작 검증
frontend/src/                           운영 화면
```

## 실행 조건과 설치

JDK 21, Node.js 20.19 이상, Docker Compose가 필요합니다. `.env.example`을 `.env`로 복사해 로컬 DB 접속 값을 채운 뒤 실행합니다. 실제 API 키는 저장소에 포함하지 않았습니다.

```bash
cp .env.example .env
docker compose --env-file .env up -d postgres
set -a; source .env; set +a
./gradlew bootRun
```

다른 터미널에서 화면을 실행합니다.

```bash
cd frontend
npm ci
npm run dev
```

화면은 `http://localhost:5173`에서 열립니다. 데이터 수집·정규화·게시 작업은 서버 시작 시 자동 실행되지 않으며, 각 작업을 명시적으로 실행하도록 분리했습니다.

## 사용 예시

서버가 실행 중이면 `curl http://localhost:8080/api/v1/overview`로 수집·게시 현황을 조회할 수 있습니다. 실제 데이터를 수집하려면 `.env`에 해당 기관의 API 키를 설정한 뒤 별도 실행에서 다음 명령을 사용합니다.

```bash
set -a; source .env; set +a
./gradlew bootRun --args='--spring.profiles.active=manual-ingestion --loanops.manual.source=FSS_FINLIFE'
./gradlew bootRun --args='--spring.profiles.active=manual-normalization --loanops.manual.run-id=<SOURCE_RUN_ID>'
./gradlew bootRun --args='--spring.profiles.active=manual-publication --loanops.manual.run-id=<SOURCE_RUN_ID>'
```

첫 명령의 결과에서 source run ID를 확인하고 뒤의 두 명령에 넣습니다. 서울시 데이터는 source를 `SEOUL_POLICY_LOAN_RATE`로 바꿔 실행합니다.

## 검증

```bash
./gradlew test
cd frontend && npm run typecheck && npm run build
```

백엔드 통합 테스트는 PostgreSQL Testcontainers와 WireMock으로 API 오류, 불완전한 페이지, 중복 정규화, 품질 실패 시 기존 게시 데이터 유지 등을 확인합니다.

## 실행 중 문제가 생기면

- DB 연결 오류: Docker가 실행 중인지, `.env`의 `POSTGRES_PASSWORD`와 `DATABASE_PASSWORD`가 같은지 확인합니다.
- 수집 실행 실패: HTTP 연결 성공과 원천 API의 성공 응답은 별개입니다. `Source Runs` 화면에서 실패 상태와 오류 유형을 확인합니다.

## 문의·라이선스·변경 이력

질문이나 오류는 [GitHub Issues](https://github.com/jmkim2021/Loanops/issues)에 남길 수 있습니다. 코드는 [MIT 라이선스](LICENSE)를 따릅니다. 변경 내역은 [커밋 기록](https://github.com/jmkim2021/Loanops/commits/main)에서 확인할 수 있습니다.

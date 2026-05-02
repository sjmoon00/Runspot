# RunSpot

> 사용자 위치 기반 러닝 그룹 매칭 서비스

[![Java](https://img.shields.io/badge/Java-17-007396?logo=openjdk)](https://openjdk.org/)
[![Spring Boot](https://img.shields.io/badge/Spring_Boot-4.0.2-6DB33F?logo=springboot)](https://spring.io/projects/spring-boot)
[![MySQL](https://img.shields.io/badge/MySQL-8.0-4479A1?logo=mysql&logoColor=white)](https://www.mysql.com/)
[![Redis](https://img.shields.io/badge/Redis-FF4438?logo=redis&logoColor=white)](https://redis.io/)
[![AWS EC2](https://img.shields.io/badge/AWS_EC2-FF9900?logo=amazonec2&logoColor=white)](https://aws.amazon.com/ec2/)

## 프로젝트 소개

RunSpot은 러닝을 즐기는 사람들이 지도에서 주변 러닝 그룹을 찾고, 참여 신청부터 출석 관리까지 한 번에 처리할 수 있는 매칭 플랫폼입니다.

- **개발 기간:** 2026.01 – 진행중
- **팀 구성:** 3인 팀 프로젝트
- **API 문서:** [Swagger UI](https://api-ide.sjm00.link/swagger-ui/index.html)

## 기술 스택

| 분류 | 기술 |
|---|---|
| Language | Java 17 |
| Framework | Spring Boot 4.0.2, Spring Security, Spring Data JPA |
| Database | MySQL 8.0, Hibernate Spatial (R-tree 공간 인덱스) |
| Cache | Redis (Lettuce) |
| ORM | QueryDSL 5.1.0 (Jakarta) |
| Infra | AWS EC2, GitHub Actions |
| Docs | SpringDoc OpenAPI (Swagger UI) |

## 주요 기능

- **세션 검색**: 이름 기반 키워드 검색 (커서 기반 무한스크롤) / 현재 위치 기준 반경 검색
- **지도 마커**: 지도 뷰포트 범위 내 세션 마커 조회 (클러스터링 최적화)
- **세션 관리**: 생성 · 마감 · 완료 · 성별 정책 적용
- **참여 플로우**: 신청 → 호스트 승인/거절 → 출석 체크
- **제재 시스템**: 노쇼 누적 시 자동 정지, 매너온도 연동

## 아키텍처

```
com.highpass.runspot
├── auth/          # 사용자 인증 및 프로필
│   ├── api/       # AuthController, UserStatsController
│   ├── domain/    # User, UserRunningStats, UserSuspensionManager
│   └── service/   # AuthService, UserStatsService
├── session/       # 러닝 세션 관리
│   ├── api/       # SessionController
│   ├── domain/    # Session, SessionParticipant
│   └── service/   # SessionService (쓰기), SessionQueryService (읽기/조회)
└── common/        # 공통 모듈
    ├── config/    # SecurityConfig, SwaggerConfig, WebConfig
    ├── exception/ # BaseException, ApiExceptionHandler
    └── util/      # GeometryUtil (JTS 좌표 유틸)
```

**읽기/쓰기 서비스 분리:** 조회 로직의 복잡도가 높아짐에 따라 `SessionService`(상태 변경)와 `SessionQueryService`(조회 전용)를 분리했습니다.

## 담당 역할

### 세션 조회 API

공간 데이터 기반 세션 검색 전체를 담당했습니다.

- 세션 이름 검색 (커서 기반 무한스크롤 페이징)
- 현재 위치 기준 반경 N km 내 세션 검색
- 지도 뷰포트 범위 기반 마커 조회
- 세션 상세보기 / 요약 정보 조회

### 배포 · 인프라

- GitHub Actions 기반 EC2 자동 배포 파이프라인 구축
- systemd 서비스 등록 및 무중단 재시작 스크립트 작성

---

## 기술적 의사결정

### 01. 공간 인덱스 타입 선택 — Geometry + Spatial Index(R-tree) 채택

**문제:** 인덱스 미적용 상태에서 도시 단위 범위 조회 시 p(95) 35,313ms — 사실상 풀스캔

**선택지 비교:**

| | Geometry + Spatial Index (R-tree) | LatLng + 복합 B-tree |
|---|---|---|
| 범위 검색 | ST_Within 폴리곤 지원 | 직사각형 근사값만 가능 |
| 확장성 | 복잡한 경로/영역 쿼리 가능 | 단순 범위 검색만 |
| 구현 복잡도 | JTS + Hibernate Spatial 필요 | 단순 float 컬럼 |

**결정:** 향후 러닝 루트 기반 검색 등 공간 쿼리 확장을 고려해 Geometry + Spatial Index 채택

**결과:** p(95) **35,313ms → 784ms** (97.8% 개선)

---

### 02. 광역 조회 4MB 페이로드 문제 — 서버 사이드 1km 격자 클러스터링 도입

**문제:** 광역 지도 조회 시 약 8만 개 마커 JSON 직렬화 → 4MB 페이로드 → Lettuce 명령 큐 포화로 연쇄 타임아웃

**선택지 비교:**

| | Redis GEO | 서버 사이드 격자 클러스터링 |
|---|---|---|
| 구현 방식 | GEORADIUS 명령으로 근접 마커 조회 | 1km 단위 격자로 {좌표, 개수} 집계 |
| 페이로드 | 마커 수에 비례 | 격자 수에 비례 (고정 크기) |
| 키 관리 | 마커 생성/삭제마다 GEO 키 업데이트 필요 | 조회 시 동적 집계 가능 |

**결정:** Redis GEO는 키 관리 복잡도가 과다하여 기각. 서버에서 1km 격자 단위로 클러스터링 후 `{좌표, 마커 개수}` 형태로 반환

**결과:** 광역 뷰 페이로드 **4MB → 40KB** (99% 감소), Redis 타임아웃 완전 해소

---

### 03. 반복 조회 DB 부하 해소 — 검색 범위별 3단계 캐싱 전략

**문제:** 동일 지역 반복 조회 시 DB 쿼리 집중. 단순 캐싱 도입 시 대용량 페이로드로 오히려 응답 시간이 11초까지 악화

**결정:** 검색 범위에 따라 캐시 전략을 세분화

| 범위 | 전략 | 이유 |
|---|---|---|
| 좁은 범위 | DB 직접 조회 | 결과 수가 적어 캐시 이점 없음 |
| 중간 범위 | 1km 타일 단위 캐시 | 타일별 분할로 페이로드 최소화 |
| 넓은 범위 | Wide 캐시 (클러스터링 결과 캐시) | 클러스터링 연산 결과 재사용 |

추가로, 세션 생성·종료 이벤트 발생 시 해당 격자 캐시를 즉시 무효화하는 이벤트 기반 전략을 적용했습니다. Redis 장애 시에는 DB로 자동 폴백하여 가용성을 확보했습니다.

**결과:** p(95) **11,000ms → 2,748ms** (75% 개선), RPS **1.59 → 21.78** (13.7배 향상)

---

## ERD

> 추후 추가 예정

## 실행 방법

```bash
# 환경변수 설정 (src/main/resources/application-secret.yml)
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/runspot
    username: {DB_USERNAME}
    password: {DB_PASSWORD}

# 빌드 및 실행
./gradlew bootRun
```

Swagger UI: `http://localhost:8080/swagger-ui/index.html`

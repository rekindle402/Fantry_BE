# Fantry Backend - Contribution Summary

## 1. 프로젝트 개요

**Fantry**는 팬덤 굿즈의 검수 요청, 거래, 고객 문의, 환불 흐름을 지원하는 경매 기반 중고 굿즈 거래 플랫폼입니다.

본 프로젝트에서 저는 고객지원(CS) 도메인 개발과 함께, 공통 기능 모듈화, 조회 성능 개선, 배포 자동화 및 운영 환경 구성을 담당했습니다.

---

## 2. 담당 범위

### 고객지원(CS) 도메인

- 1:1 문의 등록 / 조회 / 수정
- FAQ 등록 / 조회 / 수정
- 관리자 문의 조회 및 답변 처리
- 환불 / 반품 흐름 연동
- 문의 첨부파일 처리

### 공통 기능 모듈화

- 파일 업로드 / 조회 / 삭제 흐름 공통화
- JPA Specification 기반 동적 검색 조건 처리
- 공통 예외 응답 형식 구성
- 엔티티 공통 필드 분리

### 인프라·배포·운영 환경 구성

- Mini PC + Ubuntu 기반 Linux 서버 환경 구성
- DDNS, Port Forwarding, Nginx Reverse Proxy 기반 외부 요청 흐름 구성
- GitHub Actions, Shell Script, systemd 기반 배포 자동화
- Dev / Prod 환경 분리
- Promtail, Loki, Grafana 기반 로그 확인 환경 구성
- Spring Profile, GitHub Secrets, Flyway, SSH Key, DB 터널링 스크립트를 활용한 팀 개발 환경 표준화

### 조회 성능 개선

- 문의 목록 조회 API의 JPA N+1 문제 분석 및 개선
- Fetch Join 적용과 Count 쿼리 분리
- 평균 응답 속도 418ms → 61ms 개선

---

## 3. 주요 성과

| 구분 | 성과 |
|---|---|
| 조회 성능 개선 | 문의 목록 조회 API 평균 응답 속도 418ms → 61ms 개선 |
| 쿼리 최적화 | 문의 목록 조회 쿼리 수 N+1 → 1로 감소 |
| 배포 자동화 | 수동 배포 시간 약 10분 → 2~3분 단축 |
| 공통화 | 파일 처리 및 동적 검색 조건 로직을 공통화하여 도메인별 중복 구현 감소 |
| 운영 환경 | GitHub Actions, Nginx, systemd 기반 배포·실행 흐름 구성 |
| 로그 확인 | Loki/Grafana 기반 배포 후 오류 로그 확인 흐름 구성 |
| 팀 개발 환경 | 팀원별 서버 접속 및 DB 터널링 절차 표준화 |

---

## 4. 상세 기여

### 4-1. CS 도메인 개발

CS 도메인에서는 1:1 문의, FAQ, 관리자 문의 조회 및 답변 처리 기능을 담당했습니다.

문의 등록 과정에서 첨부파일 처리 흐름을 함께 고려했고, 파일 저장 로직은 공통 `FileService`를 통해 처리하도록 구성했습니다.

### 4-2. 공통 기능 모듈화

파일 처리, 동적 검색 조건, 예외 응답, 엔티티 공통 필드처럼 여러 도메인에서 반복될 수 있는 로직을 공통 영역으로 분리했습니다.

이를 통해 도메인 서비스가 비즈니스 로직에 집중할 수 있도록 하고, 공통 정책 변경 시 수정 범위를 줄이고자 했습니다.

### 4-3. 조회 성능 개선

문의 목록 조회 API에서 작성자 정보를 함께 조회하는 과정에서 N+1 문제가 발생했습니다.

Fetch Join을 적용해 N+1 문제를 해결했지만, 페이징 Count 쿼리에서 `SemanticException`이 발생했습니다.

이를 해결하기 위해 조회 쿼리와 Count 쿼리를 분기하고, 조회 쿼리에만 Fetch Join을 적용했습니다.

### 4-4. 인프라·배포·로깅 환경 구성

GitHub Actions와 Shell Script를 활용해 배포 과정을 자동화하고, systemd를 통해 애플리케이션 실행과 재시작을 관리했습니다.

또한 Nginx Reverse Proxy를 통해 Dev / Prod 요청을 분리하고, Promtail / Loki / Grafana를 활용해 배포 후 로그 확인 흐름을 구성했습니다.

---

## 5. 문서 링크

| 문서 | 설명 |
|---|---|
| `infra-deploy.md` | 인프라·배포·로깅 환경 구성 |
| `common-module.md` | 공통 기능 모듈화 |
| `performance-n-plus-one.md` | JPA N+1 및 Count 쿼리 문제 개선 |

---

## 6. 관련 코드

| 구분 | 파일 |
|---|---|
| 공통 파일 처리 | `common/util/file/*` |
| Specification 공통 추상화 | `common/util/AbstractSpecification.java` |
| 문의 Specification | `inquiry/repository/InquirySpecification.java` |
| 문의 Service | `inquiry/service/InquiryService.java` |
| 공통 예외 응답 | `common/exception/ErrorResponse.java` |
| 전역 예외 처리 | `common/exception/GlobalExceptionHandler.java` |
| 엔티티 공통 필드 | `common/domain/BaseTimeEntity.java`, `common/domain/BaseAuditingEntity.java` |

---

## 7. 정리

이 프로젝트에서는 단순 기능 구현뿐 아니라, 반복되는 서버 로직을 공통화하고, 조회 성능 문제를 분석해 개선하며, 팀 단위 개발과 배포를 위한 운영 환경을 구성하는 경험을 했습니다.

특히 공통 모듈, 성능 개선, 배포 자동화, 로그 확인 환경은 이후 서버 개발에서 기능 구현 이후의 운영 가능성과 유지보수성을 함께 고려하게 된 계기가 되었습니다.

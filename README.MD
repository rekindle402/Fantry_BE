# Fantry Backend - Contribution Summary

본 저장소는 팀 프로젝트 **Fantry Backend**의 fork 저장소이며, 포트폴리오 검토를 위해 제가 담당한 기여 내용을 중심으로 정리했습니다.

> 원본 저장소: https://github.com/SinsegeaBackend-8th-Team4/fantry-backend

---

## 프로젝트 개요

**Fantry**는 팬덤 굿즈의 검수 요청, 거래, 고객 문의, 환불 흐름을 지원하는 경매 기반 중고 굿즈 거래 플랫폼입니다.

본 프로젝트에서 저는 고객지원(CS) 도메인 개발과 함께, 공통 기능 모듈화, 조회 성능 개선, 배포 자동화 및 운영 환경 구성을 담당했습니다.

---

## 담당 범위

### 고객지원(CS) 도메인

- 1:1 문의 등록/조회/수정
- FAQ 등록/조회/수정
- 관리자 문의 조회 및 답변 처리
- 환불/반품 흐름 연동

### 공통 기능 모듈화

- 파일 업로드/조회/삭제 흐름 공통화
- JPA Specification 기반 동적 검색 조건 처리
- 공통 예외 응답 형식 구성
- 엔티티 공통 필드 분리

### 인프라·배포·운영 환경 구성

- Mini PC + Ubuntu 기반 Linux 서버 환경 구성
- DDNS, Port Forwarding, Nginx Reverse Proxy 기반 외부 요청 흐름 구성
- GitHub Actions, Shell Script, systemd 기반 배포 자동화
- Dev/Prod 환경 분리
- Promtail, Loki, Grafana 기반 로그 확인 환경 구성
- Spring Profile, Git Secrets, Flyway, SSH Key, DB 터널링 스크립트를 활용한 팀 개발 환경 표준화

### 조회 성능 개선

- 문의 목록 조회 API의 JPA N+1 문제 분석 및 개선
- Fetch Join 적용과 Count 쿼리 분리
- MyBatis 기반 조회 구조 개선 경험

---

## 주요 성과

| 구분 | 성과 |
|---|---|
| 조회 성능 개선 | 문의 목록 조회 API 평균 응답 속도 **418ms → 61ms** 개선 |
| 배포 자동화 | 수동 배포 시간 **약 10분 → 2~3분** 단축 |
| 공통화 | 파일 처리 및 동적 검색 조건 로직을 공통화하여 도메인별 중복 구현 감소 |
| 운영 환경 | GitHub Actions, Nginx, systemd 기반 배포·실행 흐름 구성 |
| 로그 확인 | Loki/Grafana 기반 배포 후 오류 로그 확인 흐름 구성 |
| 팀 개발 환경 | 팀원별 서버 접속 및 DB 터널링 절차 표준화 |

---

## 상세 문서

| 문서 | 설명 |
|---|---|
| [Contribution Summary](./docs/contribution-summary.md) | 담당 범위와 주요 기여 요약 |
| [Infra & Deploy](./docs/infra-deploy.md) | 인프라, 배포 자동화, 로그 확인 환경 구성 |
| [Common Module](./docs/common-module.md) | 파일 처리, 검색 조건, 예외 응답, 엔티티 공통 필드 모듈화 |
| [Performance Improvement](./docs/performance-n-plus-one.md) | JPA N+1 및 Count 쿼리 문제 개선 |

---

## 기술 스택

### Backend

Java, Spring Boot, Spring Data JPA, MyBatis

### Database

MySQL, Redis, Flyway

### Infra / Operation

Ubuntu, Nginx, systemd, GitHub Actions, Shell Script, SSH, DDNS

### Monitoring / Logging

Promtail, Loki, Grafana

---

## Repository Guide

이 저장소는 포트폴리오 검토를 위한 fork 저장소입니다.  
프로젝트 전체 기능보다, 제가 담당한 서버 개발·공통화·운영 환경 구성·성능 개선 경험을 중심으로 문서를 정리했습니다.

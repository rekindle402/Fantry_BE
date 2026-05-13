# 공통 기능 모듈화

## 1. 문제 상황

파일 처리, 동적 검색 조건, 예외 응답, 엔티티 공통 필드가 각 도메인에 분산될 경우 코드 중복과 수정 범위가 커질 수 있다고 판단했습니다.

특히 파일 업로드·조회·삭제 흐름은 여러 도메인에서 반복될 가능성이 있었고, 관리자/사용자 조회 화면에서는 검색 조건 조합이 늘어날 수 있었습니다.

---

## 2. 설계 방향

- 도메인 서비스는 비즈니스 로직에 집중
- 파일 저장, 파일 메타데이터 관리, 파일 URL 생성은 공통 영역으로 분리
- 검색 조건 조합 로직은 `Specification` 기반으로 구조화
- 예외 응답 형식과 엔티티 공통 필드는 프로젝트 전반에서 일관되게 사용
- 정책 변경 시 수정 범위를 공통 모듈 중심으로 제한

---

## 3. 공통화 대상

| 구분 | 공통화 내용 |
|---|---|
| 파일 처리 | 파일 업로드, 조회, 삭제, 접근 URL 생성 흐름 공통화 |
| 동적 검색 조건 | `AbstractSpecification` 기반 검색 조건 조합 로직 공통화 |
| 예외 응답 | `GlobalExceptionHandler`, `ErrorResponse` 기반 예외 응답 형식 표준화 |
| 엔티티 공통 필드 | `BaseTimeEntity`, `BaseAuditingEntity` 기반 생성/수정 시간 필드 분리 |

---

## 4. 파일 처리 공통화

### 문제

파일 업로드, 저장 경로 생성, 메타데이터 저장, 파일 URL 반환 로직이 각 도메인에 흩어질 경우 중복 구현이 늘고, 저장 정책 변경 시 수정 범위가 커질 수 있었습니다.

### 구현

파일 처리 흐름을 `FileService`와 `FileManager`로 분리했습니다.

| 구성 | 역할 |
|---|---|
| `FileService` | 파일 업로드, 파일 메타데이터 저장, 파일 URL 조회, 삭제 처리 |
| `FileManager` | 실제 물리 파일 저장, 삭제 폴더 이동, 복구, 접근 URL 생성 |
| `FileMeta` | 원본 파일명, 저장 파일명, 저장 경로, 파일 타입, 크기, 업로드 사용자, 삭제 시간 등 메타데이터 관리 |
| `FileMetaRepository` | 파일 메타데이터 조회 및 저장 |

### 처리 흐름

```text
Domain Service
  ↓
FileService
  ↓
FileManager
  ↓
Local / Server Storage

FileService
  ↓
FileMetaRepository
  ↓
filemeta table
```

### 적용 예시

`InquiryService`에서는 문의 첨부파일 저장 시 `FileService.uploadFiles()`를 호출하고, 상세 조회 시 필요한 fileMetaId 목록을 모아 `FileService.getFileAccessUrls()`를 한 번만 호출해 파일 URL을 조회합니다.

```java
List<FileMeta> savedFileMetas = fileService.uploadFiles(files, SUB_DIRECTORY, member);
savedFileMetas.forEach(inquiry::addAttachment);
```

```java
List<Integer> fileMetaIds = inquiry.getAttachments().stream()
    .map(attachment -> attachment.getFilemeta().getFilemetaId())
    .toList();

Map<Integer, String> urlMap = fileService.getFileAccessUrls(fileMetaIds);
```

### 결과

- 파일 처리 책임을 도메인 서비스에서 분리
- 파일 저장 방식과 접근 URL 생성 방식을 공통화
- 도메인 서비스는 문의, 회원, 상품 등 비즈니스 흐름에 집중 가능
- 파일 정책 변경 시 `FileService` / `FileManager` 중심으로 수정 가능

---

## 5. 동적 검색 조건 공통화

### 문제

관리자/사용자 조회 화면에서는 상태, 유형, 작성자명 등 여러 검색 조건이 조합될 수 있었습니다.

조건 조합 로직이 각 Repository나 Service에 흩어질 경우 조회 조건이 늘어날수록 코드가 복잡해질 수 있다고 판단했습니다.

### 선택 이유

QueryDSL을 추가 도입하기보다, Spring Data JPA에서 제공하는 `Specification`으로 관리자/사용자 조회의 동적 조건 조합 요구를 충족할 수 있다고 판단했습니다.

프로젝트 규모와 팀 학습 비용을 고려해, 조건 조합 로직을 `AbstractSpecification`으로 추상화하는 방향을 선택했습니다.

### 구현

`AbstractSpecification`은 공통 조건 생성 메서드를 제공합니다.

| 메서드 | 역할 |
|---|---|
| `base()` | 동적 쿼리 조합을 시작하기 위한 기본 조건 |
| `equal()` | 특정 필드의 동등 비교 조건 생성 |
| `like()` | 특정 필드의 부분 일치 검색 조건 생성 |

`InquirySpecification`은 `AbstractSpecification`을 상속해 문의 조회에 필요한 조건을 조합합니다.

| 조건 | 설명 |
|---|---|
| `status` | 문의 답변 상태 |
| `csTypeId` | 문의 유형 |
| `memberName` | 작성자 이름 부분 일치 |

### 처리 흐름

```text
InquirySearchCondition
  ↓
InquirySpecification
  ↓
AbstractSpecification
  ↓
Specification 조합
  ↓
InquiryRepository.findAll(spec, pageable)
```

### 적용 예시

```java
@Transactional(readOnly = true)
public Page<InquirySummaryResponse> searchInquires(
        InquirySearchCondition condition,
        Pageable pageable
) {
    Specification<Inquiry> spec = inquirySpecification.toSpecification(condition);

    return inquiryRepository.findAll(spec, pageable)
            .map(InquirySummaryResponse::from);
}
```

### 결과

- 검색 조건 조합 로직을 도메인별 Specification으로 분리
- 조건 추가 시 기존 Service 로직 변경 범위 감소
- 조회 조건이 늘어나도 `Specification` 조합 방식으로 확장 가능
- 관리자/사용자 조회 요구에 따라 동적 검색 조건을 재사용 가능

---

## 6. 예외 응답 형식 표준화

### 문제

도메인별 예외 응답 형식이 달라질 경우 클라이언트가 에러 응답을 처리하기 어려워지고, 서버 내부에서도 예외 응답 규칙이 일관되지 않을 수 있다고 판단했습니다.

### 구현

`GlobalExceptionHandler`에서 도메인별 커스텀 예외를 처리하고, `ErrorResponse` 형식으로 응답을 구성했습니다.

처리 대상에는 파일, 회원, 인증, 문의, 결제, 알림, 반품 등 여러 도메인 예외가 포함됩니다.

### 결과

- 도메인별 예외 응답 형식 통일
- 클라이언트가 일관된 구조로 에러 응답 처리 가능
- 처리되지 않은 예외에 대한 기본 응답 처리 흐름 마련

---

## 7. 엔티티 공통 필드 분리

### 문제

생성일, 수정일처럼 여러 엔티티에서 반복되는 필드를 각 엔티티에 개별 선언하면 중복이 발생하고, 필드 정책 변경 시 수정 범위가 커질 수 있었습니다.

### 구현

`BaseTimeEntity`, `BaseAuditingEntity`를 통해 공통 시간 필드를 분리했습니다.

| 구성 | 역할 |
|---|---|
| `BaseTimeEntity` | 생성 시간, 수정 시간 등 시간 필드 공통화 |
| `BaseAuditingEntity` | 등록자, 수정자 등 감사 관련 필드 공통화 |

### 결과

- 엔티티 공통 필드 중복 감소
- 생성/수정 시간 관리 방식 일관화
- 엔티티 설계 시 반복 필드 선언 감소

---

## 8. 결과

| 항목 | 결과 |
|---|---|
| 책임 분리 | 도메인 서비스와 공통 처리 책임 분리 |
| 파일 처리 | 업로드, 조회, 삭제, URL 생성 흐름 공통화 |
| 검색 조건 | Specification 기반 동적 검색 조건 조합 구조화 |
| 예외 응답 | 도메인별 예외 응답 형식 표준화 |
| 엔티티 설계 | 생성/수정 시간 등 공통 필드 분리 |
| 유지보수 | 공통 정책 변경 시 수정 범위 축소 |

---

## 9. 관련 코드

| 구분 | 파일 |
|---|---|
| 파일 처리 공통화 | `common/util/file/FileService.java` |
| 파일 저장 처리 | `common/util/file/FileManager.java` |
| 파일 메타데이터 | `common/util/file/FileMeta.java` |
| 파일 Repository | `common/util/file/FileMetaRepository.java` |
| Specification 공통 추상화 | `common/util/AbstractSpecification.java` |
| 문의 검색 Specification | `inquiry/repository/InquirySpecification.java` |
| 문의 검색 조건 DTO | `inquiry/dto/InquirySearchCondition.java` |
| 문의 Service 적용 예시 | `inquiry/service/InquiryService.java` |
| 공통 예외 응답 | `common/exception/ErrorResponse.java` |
| 전역 예외 처리 | `common/exception/GlobalExceptionHandler.java` |
| 엔티티 공통 필드 | `common/domain/BaseTimeEntity.java`, `common/domain/BaseAuditingEntity.java` |

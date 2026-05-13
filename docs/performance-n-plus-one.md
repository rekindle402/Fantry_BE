# JPA N+1 및 Count 쿼리 문제 개선

## 1. 문제 상황

문의 목록 조회 API에서 문의 작성자 정보를 함께 조회하는 과정에서 N+1 문제가 발생했습니다.

또한 N+1 문제 해결을 위해 Fetch Join을 적용한 뒤, 페이징 조회에서 함께 실행되는 Count 쿼리에서 500 Error가 발생했습니다.

---

## 2. 발생 지점

관리자/사용자 문의 목록 조회에서는 검색 조건에 따라 문의 목록을 페이징 조회해야 했습니다.

조회 결과에는 문의 정보뿐 아니라 작성자 정보도 함께 필요했기 때문에, 연관된 `member` 정보를 조회하는 과정에서 추가 쿼리가 반복 발생했습니다.

```text
Inquiry 목록 조회
  ↓
각 Inquiry의 member 접근
  ↓
member 조회 쿼리 반복 발생
  ↓
N+1 문제 발생
```

---

## 3. 원인 분석

### N+1 문제

JPA 지연 로딩 구조에서 문의 목록을 먼저 조회한 뒤, 각 문의의 작성자 정보를 접근하면서 member 조회 쿼리가 반복 발생했습니다.

목록 조회에서는 작성자 정보가 함께 필요했기 때문에, 조회 쿼리에서 Fetch Join을 적용해 연관 데이터를 함께 가져오도록 개선할 필요가 있었습니다.

### Count 쿼리 오류

페이징 조회에서는 실제 목록 조회 쿼리와 함께 Count 쿼리가 실행됩니다.

하지만 동일한 `Specification`이 조회 쿼리와 Count 쿼리에 함께 적용되면서, Count 쿼리에도 Fetch Join이 포함되었습니다.

Count 쿼리는 Fetch Join 대상의 owner를 select하지 않기 때문에 `SemanticException`이 발생했습니다.

---

## 4. 해결 방법

조회 쿼리와 Count 쿼리를 구분해 Fetch Join 적용 여부를 다르게 처리했습니다.

- 조회 쿼리에는 Fetch Join 적용
- Count 쿼리에는 Fetch Join 제외
- `query.getResultType()`을 기준으로 쿼리 타입 분기
- 기존 검색 조건 조합은 `Specification` 구조 안에서 유지

### 핵심 코드 예시

```java
private void applyFetchJoin(Root<Inquiry> root, CriteriaQuery<?> query) {
    Class<?> resultType = query.getResultType();

    if (resultType != Long.class && resultType != long.class) {
        root.fetch("member", JoinType.LEFT);
        query.distinct(true);
    }
}
```

---

## 5. 개선 흐름

```text
개선 전
Inquiry 목록 조회
  ↓
각 Inquiry의 member 접근
  ↓
member 조회 쿼리 반복
  ↓
N+1 발생

개선 후
조회 쿼리 여부 확인
  ↓
조회 쿼리: member Fetch Join 적용
  ↓
Count 쿼리: Fetch Join 제외
  ↓
N+1 및 Count 쿼리 오류 해결
```

---

## 6. 검증

### 쿼리 수

| 구분 | 결과 |
|---|---|
| 개선 전 | N+1 쿼리 발생 |
| 개선 후 | 조회 쿼리 1건으로 감소 |

### 응답 속도

동일 조건에서 문의 목록 조회 API의 평균 응답 속도를 비교했습니다.

| 구분 | 평균 응답 속도 |
|---|---|
| 개선 전 | 418ms |
| 개선 후 | 61ms |

### 오류 해결

| 구분 | 결과 |
|---|---|
| 개선 전 | Count 쿼리에서 `SemanticException` 발생 |
| 개선 후 | Count 쿼리에서 Fetch Join 제외 후 정상 동작 |

---

## 7. 결과

- 문의 목록 조회 API의 N+1 문제 해결
- 쿼리 수 N+1 → 1로 감소
- 평균 응답 속도 418ms → 61ms 개선
- 페이징 Count 쿼리 오류 해결
- 검색 조건 조합 구조를 유지하면서 조회 성능 개선

---

## 8. 관련 코드

| 구분 | 파일 |
|---|---|
| Fetch Join 분기 및 검색 조건 조합 | `inquiry/repository/InquirySpecification.java` |
| 문의 검색 조건 DTO | `inquiry/dto/InquirySearchCondition.java` |
| 문의 조회 Service | `inquiry/service/InquiryService.java` |
| 문의 Repository | `inquiry/repository/InquiryRepository.java` |
| Specification 공통 추상화 | `common/util/AbstractSpecification.java` |

---
## 9. 검증 기준

동일한 조회 조건에서 문의 목록 조회 API의 평균 응답 속도를 비교했습니다.

| 구분 | 결과 |
|---|---|
| 개선 전 | 평균 418ms |
| 개선 후 | 평균 61ms |
| 쿼리 수 | N+1 → 1 |
| 오류 | Count 쿼리 SemanticException 해결 |

※ 응답 속도는 포트폴리오 PDF의 측정 캡처를 기준으로 정리했습니다.

## 10. 개선하면서 배운 점

단순히 Fetch Join을 적용하는 것만으로는 페이징 조회 문제를 모두 해결할 수 없었습니다.

목록 조회와 Count 쿼리는 목적이 다르기 때문에, ORM을 사용할 때는 실제 실행되는 쿼리의 종류와 조회 목적을 함께 고려해야 한다는 점을 배웠습니다.

# ADR-012: 도메인 검증 클래스 설계 — DomainValidator<T> 도입

## 상태
승인됨

## 날짜
2026-04-28

## 맥락

견적(Quote) 저장/수정 시 거래처, 담당자, 결재자, 품목 등 FK 참조 데이터의 존재 여부를 검증해야 한다.
초기 구현에서는 `QuoteValidator`가 Spring의 `Validator` 인터페이스를 구현했으나,
`supports()`와 `validate(Errors)` 메서드를 비워 둔 채 자체 메서드(`validateHeader`, `validateLines`)만 호출하는
형태여서 인터페이스 계약을 실질적으로 위반하고 있었다.

검증 클래스가 여러 도메인(salesorder, bom, routing 등)으로 확장될 예정이므로,
일관된 구조를 강제할 수 있는 패턴을 사전에 결정한다.

## 검토한 선택지

- **선택지 A: Spring `Validator` + `WebDataBinder` (컨트롤러 레이어)** — `@InitBinder`로 컨트롤러에 검증기를 등록하고 `MethodArgumentNotValidException`으로 통일한다.
- **선택지 B: Spring `Validator` + 서비스 레이어 직접 호출** — `BeanPropertyBindingResult`를 서비스에서 직접 생성해 `Errors`에 누적 후 확인한다.
- **선택지 C: 커스텀 `DomainValidator<T>` 인터페이스** — 프로젝트 예외 컨벤션에 맞게 직접 예외를 던지는 방식으로 인터페이스를 정의한다.

## 결정

**선택지 C — 커스텀 `DomainValidator<T>` 인터페이스**를 채택한다.

선택지 A를 기각한 이유:
- FK 존재 여부 검증은 DB 조회가 필요한 **비즈니스 검증**으로, 입력값 형식 검증(`@Valid`)과 성격이 다르다.
- `WebDataBinder`는 컨트롤러 레이어에서 실행되어 서비스의 `@Transactional` 경계 밖에 위치한다.
- `@InitBinder`는 컨트롤러 단위로 등록되어 의도치 않은 검증이 붙을 수 있다.

선택지 B를 기각한 이유:
- `BeanPropertyBindingResult`를 서비스에서 직접 다루는 코드가 장황하다.
- 프로젝트 전체가 예외 기반(`ResourceNotFoundException`)으로 통일되어 있는데, `Errors` 누적 방식과 혼재하면 에러 처리 흐름이 분산된다.

인터페이스 정의 (`common/validation/DomainValidator.java`):

```java
public interface DomainValidator<T> {
    void validate(T request);
}
```

구현 예시:

```java
@Component
@RequiredArgsConstructor
class QuoteValidator implements DomainValidator<QuoteRequest> {

    @Override
    public void validate(QuoteRequest request) {
        validateHeader(request);
        validateLines(request.lines());
    }

    private void validateHeader(QuoteRequest request) { ... }
    private void validateLines(List<QuoteLineRequest> lines) { ... }
}
```

서비스에서의 호출:

```java
quoteValidator.validate(request);
```

## 결과

### 긍정적
- 모든 도메인 검증 클래스가 동일한 인터페이스를 구현하므로 구조가 일관됨
- 예외 기반 에러 처리(`ResourceNotFoundException`)와 자연스럽게 통합됨
- 서비스 레이어에 위치하여 `@Transactional` 경계 안에서 실행됨
- 서비스에서 `quoteValidator.validate(request)` 한 줄로 호출이 단순함

### 부정적 / 트레이드오프
- Spring `Validator`를 사용하지 않으므로 `MethodArgumentNotValidException` 체계와 통합되지 않음
  (단, 프로젝트가 예외 기반으로 통일되어 있어 실질적 문제 없음)
- 입력값 형식 검증(`@Valid`)과 비즈니스 검증이 서로 다른 메커니즘으로 처리됨 — 이는 의도된 설계

## 관련 문서
- 관련 ADR: [ADR-003](./003-dto-as-java-record.md)

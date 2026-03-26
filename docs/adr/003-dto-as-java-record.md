# ADR-003: DTO를 Java Record로 전환

## 상태
<!-- 제안됨 | 검토중 | 승인됨 | 거부됨 | 폐기됨 | 대체됨 -->
승인됨

## 날짜
2026-03-26

## 맥락
현재 프로젝트의 DTO는 49개 클래스로, 모두 `api/dto` 패키지 하위에 위치한다.
대부분이 데이터 운반이 주목적인 단순 구조임에도 Lombok 애노테이션(`@Getter`, `@Setter`,
`@NoArgsConstructor`, `@AllArgsConstructor`)을 조합해 작성되어 있다.

Java 16에서 정식 도입된 Record는 불변 데이터 캐리어에 최적화된 언어 구조로,
현재 프로젝트 환경(Java 17, Spring Boot 4, Jackson 2.17+)에서 완전히 지원된다.
`CurrentUserDto`는 이미 Record로 작성되어 있어 사실상 도입이 시작된 상태다.

## 검토한 선택지

- **선택지 A: 현행 유지** — Lombok 기반 class DTO를 그대로 사용한다.
- **선택지 B: 전체 Record 전환** — 모든 DTO를 Java Record로 전환한다.
- **선택지 C: 점진적 전환** — 신규 DTO는 Record로 작성하고, 기존 DTO는 도메인 단위로 순차 전환한다.

## 결정

**선택지 C — 점진적 전환**을 채택한다.

전환 우선순위:

1. **단순 Request DTO** (Lombok 3종 + Validation만 있는 것) — 변환이 기계적이고 리스크 없음
2. **Response DTO** (`fromRecord` 팩토리 포함) — Record body 안에 정적 메서드 유지 가능
3. **기본값이 있는 Request DTO** (`useYn = true` 등) — compact constructor로 처리

전환하지 않는 대상:

- 인터페이스 구현이 필요한 DTO (Record는 상속/구현 불가 — 현재 해당 없음)

## 결과

### 긍정적
- Lombok 의존 제거로 컴파일 타임 코드 생성 없이 동일한 기능
- 불변성 보장 — Request DTO가 컨트롤러 이후 변경될 수 없음
- `equals`, `hashCode`, `toString` 자동 생성으로 테스트 가독성 향상
- 코드량 감소 — DTO당 평균 3~4줄 절약

### 부정적 / 트레이드오프
- Record 컴포넌트에 Javadoc 작성 위치가 어색함 (필드 주석 대신 파라미터 주석)
- 기본값 처리 시 compact constructor 문법이 다소 낯설 수 있음
- 기존 class DTO와 Record DTO가 혼재하는 전환 기간 발생

## 관련 문서
- 관련 ADR: [ADR-001](./001-bom-ui-structure.md)

# Testing Principles

이 문서는 프로젝트의 테스트 작성 원칙과 패턴을 정리한 가이드입니다.  
목표는 다음 4가지입니다.

1. 도메인 규칙의 안정성을 확보한다.
2. 가치있는 테스트를 작성하여 비즈니스 로직의 정합성을 확보한다.
3. 회귀 이슈를 방지한다.

---

## 1. 테스트 레이어(피라미드) 설계

본 프로젝트는 테스트를 **Unit → Integration** 레이어로 분리한다.

### 1.1 Unit Tests (`test/unit/**`)

**목표**

- 도메인/비즈니스 규칙을 결정적으로 검증한다.
- DB/Redis 등 외부 리소스 없이 동작해야 한다.
- 1+1 = 2 수준의 가치없는 테스트는 만들지 않는다. 코어 비즈니스 규칙만을 테스트한다.

**특징**

- 엔티티 생성자/행위 메서드에서 불변식(invariant) 위반 시 예외가 발생하는지 검증한다.
- 외부 의존성은 `jest.fn()` 기반 mock을 사용하고, 테스트에서 **명시적으로 주입**한다.

### 1.2 Integration Tests (`test/integration/**`)

**목표**

- 실제 인프라와 연결된 흐름을 검증한다. (DB/Redis/Kafka 등)
- 특히 재고/잔액 같은 **정합성(Consistency)**, **동시성(Concurrency)** 요구사항을 실제 환경에 가깝게 테스트한다.

**특징**

- 단순 CRUD 검증보다, “동시에 N명이 실행했을 때 결과가 정확한가” 같은 시나리오를 중요하게 다룬다.
- 테스트 간 오염을 막기 위해 `beforeEach`에서 데이터/키를 정리(cleanup)한다.
- “모킹 없이 실제 함수를 호출하는 검증(No Mock)”을 우선하기로 한다.

### 1.3 E2E Tests

- E2E 테스트는 하지 않는다.

---

## 2. 파일/네이밍 컨벤션

### 2.1 파일 위치

- Unit: `test/unit/**`
  - domain entity : `test/unit/domain/**`
- Integration: `test/integration/**`
  - service : `test/integration/services/**`
  - use-case : `test/integration/use-cases/**`
  - repository : `test/integration/repositories/**`

### 2.2 파일명

- Unit: `*.unit.spec.ts`
- Integration: `*.integration.spec.ts`

---

## 3. 테스트 코드 구조(기본 템플릿)

테스트는 **대상 → 메서드/상황 → 케이스**로 중첩한다.

- `describe('<대상>', () => { ... })`
- `describe('<메서드/상황>', () => { ... })`
- `it('<기대 동작>', () => { ... })`

또한 테스트 본문은 **Given / When / Then**(AAA)로 구분한다.

```ts
describe('SomeDomainEntity', () => {
  describe('someBehavior', () => {
    it('유효한 입력이면 기대한 결과를 반환한다', () => {
      // given
      // when
      // then
    })
  })
})
```

> 원칙: 테스트는 “읽는 사람이 시나리오를 바로 따라갈 수 있게” 작성한다.

---

## 4. 도메인 테스트 원칙 (Unit Test 중심)

### 4.1 도메인 불변식은 Domain에서 강제한다

- 값 범위(음수 금지, 정수 강제, 소유권 검증 등)는 Domain에서 막고,
- 테스트는 “정상 케이스 + 경계값 + 실패 케이스”를 함께 다룬다.

### 4.2 예외는 타입만이 아니라 “에러 코드”까지 검증한다

- `DomainException` 또는 프로젝트 표준 예외 타입을 기대한다.
- 가능하면 `errorCode`, `ErrorMessage`까지 검증한다.

---

## 5. 모킹/대체 전략

### 5.1 Unit: 외부 의존성은 최소 단위로 mock 처리

- `jest.fn()` 기반 mock을 사용한다.
- DI 컨테이너에 의존하기보다, 테스트에서 **직접 주입**(필드 주입/생성자 주입 등)하여 의도를 드러낸다.

**권장**

- 네트워크 호출/IO는 unit test에서 금지한다.
- mock은 “행동 검증(호출 여부/파라미터)”과 “결과 유도(반환값/예외)”만 다룬다.

### 5.2 Integration: “가능한 한 실제”를 우선한다

- Redis/Kafka/DB는 테스트 전용 환경에서 실제로 연결해 동작을 검증한다.

---

## 6. 상태 격리(테스트 독립성)

### 6.1 Integration 테스트의 기본 규칙

- `beforeEach`에서 DB/Redis 등 상태를 정리한다.
- Redis는 prefix 기반 key 탐색 후 삭제하는 등 “테스트가 남긴 흔적”을 제거한다.
- DB는 `cleanupDatabase` 같은 유틸을 통해 테이블을 초기화한다.

> 원칙: 어떤 테스트도 “이전 테스트가 남긴 데이터”에 의존하면 안 된다.

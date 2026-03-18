---
applyTo: "**/*.ts"
---

# 개발 규칙

## 공통

**주석**

- 메서드에는 ANCHOR, NOTE, TODO 와 TSDOC 주석을 활용하여 명세, 설명, TODO 등을 작성합니다. 주석이 무가치한 기능은 `ANCHOR 의도설명` 한 줄로 작성하려 노력합니다.

```typescript
/** ANCHOR: 사용자 생성
 * @description: 새로운 사용자를 생성합니다.
 */
async createUser(...) {
    ...
}
```

**ORM 사용**

- `src/common/database/transaction-manager.service.ts`를 활용하며, typeorm의 dataSource를 절대 직접적으로 사용하지 않는다.
- txManager.runInTransaction은 유즈케이스에서만, txManager.client는 반드시 레포지토리에서만 활용한다.
- queryBuilder, relations를 사용하지 않고 ORM 기능만 활용한다.
- N+1 문제는 `findByIds`로 해결한다.
- 동시성 제어는 pessimistic lock을 활용한다.

**Logger 사용**
console 대신 NestJS의 Logger를 활용하여 추적을 최대한 편하게 할 수 있도록 한다.

### 개발 문화

- 공통 : 백엔드는 코드 분량을 극단적으로 줄이는 것을 목표로 하며, BACKEND_ARCHITECTURE.md에 명세된 원칙을 최대한 지키는 것을 최선을 다해 노력합니다.
- 경량화 노력 : 1 기능 1 use-case 원칙을 가지지만, 아주 간단한 API는 controller에서 도메인 서비스를 직접 호출할 수 있다.

### 폴더 구조 샘플

```
src/
├── api/
│   ├── order/
│   │   ├── presentation/
│   │   │   ├── order.controller.ts
│   │   │   └── admin.order.controller.ts
│   │   ├── application/
│   │   │   ├── create-order.use-case.ts
│   │   │   └── admin.create-order.use-case.ts
│   │   ├── order.domain.service.ts  // order-item, order-history 등은 agregate root인 order 도메인 서비스에서 함께 처리한다.
│   │   ├── order.repository.ts
│   │   ├── order-item.repository.ts
│   │   ├── order-history.repository.ts
│   │   └── dto/
│   │       ├── create-order.dto.ts
│   │       └── admin.update-order.dto.ts
│   └── ...
├── common/
│   ├── database/
│   │   └── transaction-manager.service.ts
│   └── ...
└── entities/
    ├── order.entity.ts
    ├── order-item.entity.ts
    └── order-history.entity.ts

```

### 0. DTO (공통)

- `./dto/*.dto.ts`: Request/Response DTO
- `class-transformer`, `class-validator` 사용
- 입력값 유효성 검증, presentation layer 및 application layer에서 공통으로 사용
- 마스킹이 불필요한 경우에는 가급적 Domain Entity를 그대로 전달값으로 활용하려 노력합니다.

### 1. Presentation Layer (Controller)

**역할:** HTTP 요청/응답 처리, DTO 변환, API 문서화
**구성 요소:**

presentation 계층은 유저와 관리자를 구분합니다.

- `./presentation/*.controller.ts`: 사용자 관련 라우팅
- `./presentation/admin.*.controller.ts`: 어드민 관련 라우팅

**특징:**

- Swagger 데코레이터로 API 문서 자동 생성
- 컨트롤러에서 useCase 호출을 하지만, 아주 단순한 조회/수정 등의 기능은 useCase를 만들지 않고 도메인 서비스를 직접 호출하여 코드 분량 감소를 노력한다.
- DTO를 그대로 활용하여 UseCase에 전달하여 코드 분량 감소를 노력한다.
- 트랜잭션이나 동시성 제어가 필요한 경우, 반드시 UseCase에서 트랜잭션 경계를 책임진다.

### 2. Application Layer (애플리케이션 계층)

**역할:** 유즈케이스 실행, 트랜잭션 경계, 도메인 서비스 조합

**구성 요소:**

use-case 계층은 유저와 관리자를 구분합니다.

- `application/*.use-case.ts`: 단일 유저 비즈니스 유즈케이스
- `application/admin.*.use-case.ts`: 단일 어드민 비즈니스 유즈케이스

**설계 원칙:**

- 1 UseCase = 1 기능
- 도메인 서비스와 도메인 엔티티(read-only로 취급)들을 활용하여 비즈니스 로직을 구현합니다.
- 트랜잭션 경계를 책임지며, txManager.runInTransaction을 활용하여 트랜잭션 컨텍스트를 유지합니다.
- 트랜잭션 경계가 없거나, 복잡도가 낮은 기능은 굳이 UseCase로 분리하지 않고, Controller에서 도메인 서비스를 직접 호출할 수 있다.
- 가급적 input/output은 controller에서 사용하는 DTO를 공통으로 활용한다.

```
  constructor(private readonly txManager: TransactionManager) {}
  private get txClient() {
    return this.txManager.client
  }
  someUseCase(...) {
    return this.txManager.transaction(async () => {
      // 트랜잭션 컨텍스트 내에서 도메인 서비스 호출
      await this.someDomainService.someMethod(...)
    })
  }
```

### 3. Domain Entity

**역할** 리치 도메인 모델 정의, 비즈니스 규칙 구현, Validation rules 명세, 상태 전이 명세

**구성 요소:**

- `*.entity.ts` : typeorm의 DB entity를 확장하여 리치 도메인 모델을 구현한 것을 도메인 엔티티로 활용합니다.

**특징**

- static factory method로 객체 생성을 통제하여, 객체 생성 시점에 필요한 validation과 비즈니스 규칙을 강제합니다.
- 상태 전이, 비즈니스 규칙과 Validation은 도메인 엔티티에 최대한 몰아 넣어 구현합니다.
- 상태 변경을 도메인 서비스나 유즈케이스에서 직접적으로 하는 것을 지양합니다. 도메인 엔티티(read-only로 취급)의 메서드를 통해 상태를 변경하도록 책임집니다.
- "전체 속성 setter"는 지양하고, 비즈니스 규칙 명세를 우선시하며 개별 필드나 작은 단위부터 setter를 작성하며 명시적으로 네이밍된 메서드로 구현합니다.
- 삭제 관련 규칙은 repository의 관심사이므로 작성하지 않습니다.

### 4. Domain Service (도메인 서비스)

**역할** repository 래퍼, 복합 도메인 로직 구현

**구성 요소:**

- `*.domain.service.ts`

**특징**

- **Aggregate Root** 를 책임집니다. 1개의 domain service가 다수의 repository, domain entity를 포함합니다. (예: order 도메인 서비스에서 order-item, order-history 등도 함께 처리)
- 도메인 서비스는 READ 함수에서 "리스트, 복수"를 찾으려면 `findSomething`을 쓰고, "객체, 단수"를 찾으려면 `getSomething`을 쓰도록 합니다.
- 네이밍 규칙 : 기본 CRUD 수준의 메서드 명은 Repository와 동일하게 작성하되, 복합 도메인 로직이 필요한 경우 명시적으로 네이밍된 메서드를 작성하여 구현합니다.
- Repository DIP는 활용하지 않습니다.

### 5. Repository (레포지토리)

**역할** 영속성 처리 (DB만 사용)

**구성 요소:**

- Repository: `*.repository.ts`

**특징**

- 다수의 READ 함수가 상단에 먼저 몰아서 작성되고, save, delete(soft delete) 를 하단에 작성합니다. READ 함수에서 "리스트, 복수"를 찾으려면 `findSomething`을 쓰고, "객체, 단수"를 찾으려면 `getSomething`을 쓰도록 합니다.
- update orm을 금지하고, 수정된 객체를 직접 save합니다.
- 비관적 잠금 조회는 getSomeByIdForUpdate로 네이밍하여 MVCC로 일반 조회하는 것과 구분합니다.
- dataSource 직접 사용을 금기시하고, txManager, txClient를 활용하여 트랜잭션 컨텍스트를 코드베이스 전체적으로 유지합니다.
- query builder는 타입 안정성을 위해 가급적 지양하고, find/findOne/findByIds 등 제공되는 메서드 활용을 우선시합니다.
- Repository에 조건부 where을 통째로 넣는것을 지양하고, 명시적으로 네이밍 된 메서드를 활용합니다.
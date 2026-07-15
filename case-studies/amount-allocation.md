---
layout: default
title: 수납·결제 금액 통합 분배 엔진
description: 승인, 반환, 여정변경 등 서로 다른 철도 결제 상황에서 수납정보와 결제정보를 하나의 모델로 정규화하고 금액·세금·수수료를 동일한 방식으로 분배한 설계 사례입니다.
body_class: case-study-page
---

<article class="case-study">

  <header class="case-study-header">
    <p class="case-study-header__eyebrow">
      Architecture Case Study 02
    </p>

    <h1 class="case-study-header__title">
      서로 다른 승인·반환 거래를<br>
      하나의 금액 분배 흐름으로 처리하기
    </h1>

    <p class="case-study-header__description">
      철도 발매에는 일반 카드와 현금 결제뿐 아니라 간편결제,
      후급, 계좌결제, 비상발매, 여정변경 등 다양한 거래 형태가 존재합니다.
    </p>

    <p class="case-study-header__description">
      거래 유형마다 조회해야 하는 요청정보와 기존 결제정보는 달랐지만,
      각각 다른 계산식을 적용하면 동일한 금액이 상황에 따라 다르게 분배될 수 있었습니다.
      이를 방지하기 위해 필요한 정보를 하나의 금액 모델로 변환하고,
      승인·반환 여부와 관계없이 동일한 분배 메서드를 사용하도록 설계했습니다.
    </p>

    <div class="case-study-header__tags" aria-label="관련 주제">
      <span>Amount Allocation</span>
      <span>Domain Normalization</span>
      <span>Tax Distribution</span>
      <span>Fee Allocation</span>
      <span>Fail Fast</span>
    </div>
  </header>

  <section class="case-study-summary" aria-labelledby="summary-title" markdown="1">

## 한눈에 보기 {#summary-title}

<div class="case-study-summary__grid">
  <div class="case-study-summary__item">
    <strong>문제</strong>
    <p>
      승인, 반환, 여정변경에 따라 조회해야 하는 수납정보,
      요청정보와 기존 결제정보가 달랐습니다.
    </p>
  </div>

  <div class="case-study-summary__item">
    <strong>설계</strong>
    <p>
      서로 다른 정보를 공통 금액 모델로 변환하고,
      약 14개 결제수단의 우선순위에 따라 동일한 메서드에서 분배했습니다.
    </p>
  </div>

  <div class="case-study-summary__item">
    <strong>정합성</strong>
    <p>
      결제금액뿐 아니라 공급가액, 부가세, 면세금액과 수수료까지
      각 결제 매핑 행에 정확히 귀속되도록 검증했습니다.
    </p>
  </div>
</div>
</section>

  <section class="case-study-section" aria-labelledby="background-title" markdown="1">

## 1. 배경 {#background-title}

철도 발매 시스템에는 일반적인 승인과 반환 외에도
다양한 형태의 결제 상황이 존재합니다.

- 일반 카드 및 현금 결제
- 간편결제
- 후급 처리
- 계좌결제
- 비상발매
- 단말기에서만 결제된 금액
- 기존 승차권 반환과 신규 승인이 동시에 발생하는 여정변경
- 거리와 승차 조건에 따라 달라지는 할인
- 쿠폰과 각종 할인 적용
- 반환 과정에서 발생하는 수수료

모든 할인과 쿠폰을 적용하면 고객이 실제로 납부해야 하는
최종 금액이 결정되고, 이 결과는 수납정보에 저장됩니다.

하지만 하나의 수납금액이 반드시 하나의 결제수단과 일치하지는 않습니다.

예를 들어 최종 수납금액이 30,000원이라도
실제 결제정보는 다음과 같이 구성될 수 있습니다.

```text
후급             15,000원
카카오페이        10,000원
카드               4,000원
현금               1,000원
────────────────────────
수납금액          30,000원
```

결제 매핑 테이블에는 각 수단의 결제금액뿐 아니라
수납정보에 기록된 공급가액, 부가세, 면세금액과
업무상 발생한 수수료도 함께 연결되어야 했습니다.
</section>

  <section class="case-study-section" aria-labelledby="case-title" markdown="1">

## 2. 거래 유형에 따라 달라지는 입력정보 {#case-title}

금액 분배에 필요한 정보는 거래 유형마다 달랐습니다.

### 승인

신규 승인에서는 기존 결제내역이 존재하지 않습니다.

수납정보와 이번 승인 요청에 포함된 결제수단별 금액을 기준으로
새로운 결제 매핑정보를 만들어야 합니다.

```text
수납정보
+ 신규 승인 요청정보
→ 신규 결제 매핑정보
```

### 반환

반환에서는 이미 승인 당시 생성된 결제정보가 존재합니다.

따라서 이번 반환 요청금액만 확인해서는 안 되고,
기존에 어떤 결제수단으로 얼마가 승인되었는지 확인해야 합니다.

```text
수납정보
+ 기존 승인 결제정보
+ 기존 승인 분배결과
+ 이번 반환 요청정보
+ 기존 반환내역
→ 반환 매핑정보
```

특정 결제수단으로 승인된 금액보다
더 많은 금액을 해당 수단에서 반환할 수는 없습니다.

```text
결제수단별 누적 반환금액
≤ 결제수단별 승인금액
```

### 여정변경

여정변경은 기존 승차권의 반환과
변경된 승차권의 신규 승인이 함께 발생합니다.

따라서 기존 결제정보와 신규 요청정보를 모두 확인해야 합니다.

```text
기존 수납 및 결제정보
+ 기존 승인 분배결과
+ 반환 대상정보
+ 신규 수납정보
+ 신규 승인 요청정보
→ 반환 및 신규 승인 매핑정보
```

하나의 업무 요청 안에서도 반환과 승인이 동시에 존재하므로,
각 계산을 독립적인 임시 로직으로 처리하면
금액 관계를 추적하기 어려워질 수 있었습니다.
</section>

  <section class="case-study-section" aria-labelledby="problem-title" markdown="1">

## 3. 핵심 문제 {#problem-title}

금액 분배 과정에서는 상황에 따라
4~5가지의 요청 및 DB 정보를 함께 비교해야 했습니다.

대표적으로 다음 정보가 사용됩니다.

1. 수납정보
2. 현재 승인 또는 반환 요청정보
3. 기존 결제정보
4. 기존 승인 당시의 분배결과
5. 기존 반환 및 수수료 처리내역

각 정보가 가진 관점도 서로 다릅니다.

### 수납정보

수납정보는 회계적인 전체 금액을 가지고 있습니다.

```text
전체 수납금액
과세 공급가액
부가세
면세금액
수수료
할인 적용 결과
```

### 승인·반환 요청정보

요청정보에는 실제 처리해야 할 금액과
결제수단 정보가 존재합니다.

```text
처리 구분
결제수단
승인 또는 반환 요청금액
외부 거래 식별정보
```

하지만 각 결제수단에 귀속할 공급가액이나
부가세 금액은 포함되어 있지 않았습니다.

### 기존 결제정보

반환과 여정변경에서는
기존에 어떤 수단으로 얼마가 승인되었는지를 나타냅니다.

```text
결제수단별 승인금액
기존 승인 거래 식별정보
기존 분배 공급가액
기존 분배 부가세
누적 반환금액
```

이 정보들을 계산하여 최종적으로 다음 매핑을 만들어야 했습니다.

```text
수납정보
+ 현재 요청정보
+ 기존 결제정보
+ 기존 분배정보
+ 반환 및 수수료정보
│
▼
결제수단별 매핑정보
- 승인 또는 반환금액
- 공급가액
- 부가세
- 면세금액
- 수수료
```
</section>

  <section class="case-study-section" aria-labelledby="risk-title" markdown="1">

## 4. 거래 유형별 계산 로직을 분리할 때의 위험 {#risk-title}

승인, 반환, 여정변경마다 별도의 분배 메서드를 만들면
각 로직이 조금씩 다른 기준을 사용하게 될 가능성이 컸습니다.

```text
승인용 금액 분배
반환용 금액 분배
비상발매용 금액 분배
여정변경용 반환 분배
여정변경용 신규 승인 분배
```

이 방식에서는 다음 문제가 발생할 수 있습니다.

- 승인과 반환의 결제수단 우선순위가 달라짐
- 같은 수납금액을 서로 다른 방식으로 계산
- 반환 시 기존 승인금액을 초과
- 공급가액과 부가세의 분배 기준이 달라짐
- 수수료를 일부 거래에서만 누락
- 단수 보정 방식이 거래별로 달라짐
- 결제수단 추가 시 여러 분배 로직을 모두 수정
- 승인 당시의 분배결과와 반환 결과를 비교하기 어려움

따라서 거래 유형별로 계산 메서드를 만드는 대신,
입력정보만 공통 모델로 변환한 뒤
동일한 분배 엔진을 사용하도록 구성했습니다.
</section>

  <section class="case-study-section" aria-labelledby="goal-title" markdown="1">

## 5. 설계 목표 {#goal-title}

### 거래 유형과 분배 계산을 분리한다

승인인지 반환인지 판단하고 필요한 정보를 조회하는 책임과,
실제 금액을 분배하는 책임을 분리합니다.

### 모든 거래를 공통 금액 모델로 변환한다

원본 요청정보의 형태가 다르더라도
분배 엔진에는 동일한 구조의 금액정보를 전달합니다.

### 승인 당시의 분배결과를 반환 계산의 기준으로 사용한다

반환 시 현재 수납금액만 새로 나누지 않고,
기존 승인 당시 어떤 수단에 얼마가 귀속되었는지를 기준으로 계산합니다.

### 동일한 분배 메서드를 사용한다

승인, 반환, 여정변경 여부에 따라
다른 계산 메서드를 호출하지 않습니다.

### 결제금액과 회계 구성 금액을 함께 분배한다

결제금액뿐 아니라 공급가액, 부가세,
면세금액과 수수료도 동일한 결과에 포함합니다.

### 잘못된 계산은 저장 전에 차단한다

승인금액을 초과한 반환,
합계 불일치, 음수 금액은 DB 저장 이전에 예외로 처리합니다.
</section>

  <section class="case-study-section" aria-labelledby="architecture-title" markdown="1">

## 6. 전체 구조 {#architecture-title}
```text
거래 요청
│
▼
거래 유형 판별
DB 조회를 통해 승인 / 반환 / 여정변경 확인
│
▼
AllocationContextAssembler
│
├── 수납정보 조회
├── 현재 요청정보 변환
├── 기존 결제정보 조회
├── 기존 승인 분배결과 조회
└── 반환 및 수수료내역 조회
│
▼
AllocationContext
공통 금액 모델
│
▼
AmountAllocator.allocate()
│
├── 결제수단 우선순위 정렬
├── 승인 또는 반환 가능금액 계산
├── 결제금액 분배
├── 공급가액·부가세·면세금액 분배
├── 수수료 분배
└── 전체 불변식 검증
│
▼
PaymentAmountMapping
```

어떤 거래에서도 실제 계산은
`AmountAllocator.allocate()` 하나를 사용합니다.

차이가 있는 부분은 계산 전에
어떤 정보를 조회하고 공통 모델에 넣느냐입니다.
</section>

  <section class="case-study-section" aria-labelledby="context-title" markdown="1">

## 7. 공통 금액 모델 {#context-title}

아래 코드는 실제 업무 구조를 설명하기 위해
클래스명과 일부 필드를 일반화한 예시입니다.

```java
public record AllocationContext(
TransactionType transactionType,
ReceiptAmount receipt,
List<RequestedPayment> requestedPayments,
List<ApprovedPayment> approvedPayments,
  List<PreviousReturn> previousReturns,
    FeeAmount fee
    ) {

    public record ReceiptAmount(
    BigDecimal totalAmount,
    BigDecimal taxableSupplyAmount,
    BigDecimal vatAmount,
    BigDecimal taxFreeAmount
    ) {
    }

    public record RequestedPayment(
    PaymentMethod method,
    BigDecimal amount,
    int priority
    ) {
    }

    public record ApprovedPayment(
    PaymentMethod method,
    BigDecimal approvedAmount,
    BigDecimal taxableSupplyAmount,
    BigDecimal vatAmount,
    BigDecimal taxFreeAmount
    ) {
    }

    public record PreviousReturn(
    PaymentMethod method,
    BigDecimal returnedAmount
    ) {
    }

    public record FeeAmount(
    BigDecimal totalAmount
    ) {
    }
    }
```

    실제 거래 유형에 따라 일부 정보는 비어 있을 수 있습니다.

```text
승인
- receipt              존재
- requestedPayments    존재
- approvedPayments     없음
- previousReturns      없음 또는 빈 값

반환
- receipt              존재
- requestedPayments    존재
- approvedPayments     존재
- previousReturns      존재 가능

여정변경
- 기존 거래용 context
- 신규 거래용 context
- 반환과 승인을 연속된 동일 분배 흐름으로 처리
```

    계산 엔진은 원본 요청정보나 DB 엔티티를 직접 알지 않고,
    정규화된 `AllocationContext`만 사용합니다.
</section>

  <section class="case-study-section" aria-labelledby="assembler-title" markdown="1">

## 8. 거래 유형에 맞는 정보 조립 {#assembler-title}

공통 모델을 만드는 조립기는
DB 조회 결과를 기준으로 거래 유형을 판단합니다.

```java
@Component
public class AllocationContextAssembler {

public AllocationContext assemble(
PaymentRequest request
) {
TransactionType transactionType =
transactionTypeResolver.resolve(request);

return switch (transactionType) {
case APPROVAL ->
assembleApproval(request);

case RETURN ->
assembleReturn(request);

case JOURNEY_CHANGE ->
assembleJourneyChange(request);
};
}
}
```

### 승인정보 조립

```java
private AllocationContext assembleApproval(
PaymentRequest request
) {
ReceiptAmount receipt =
receiptRepository.getRequired(request.receiptId());

List<RequestedPayment> requestedPayments =
requestedPaymentConverter.convert(request);

return AllocationContext.forApproval(
receipt,
requestedPayments,
resolveFee(request)
);
}
```

### 반환정보 조립

```java
private AllocationContext assembleReturn(
PaymentRequest request
) {
ReceiptAmount receipt =
receiptRepository.getRequired(request.receiptId());

List<ApprovedPayment> approvedPayments =
  paymentRepository.findApprovedPayments(
  request.originalTransactionId()
  );

  List<PreviousReturn> previousReturns =
    returnRepository.findPreviousReturns(
    request.originalTransactionId()
    );

    List<RequestedPayment> requestedPayments =
      requestedPaymentConverter.convert(request);

      return AllocationContext.forReturn(
      receipt,
      requestedPayments,
      approvedPayments,
      previousReturns,
      resolveFee(request)
      );
      }
```

      반환에서는 기존 승인정보와 기존 반환내역을 함께 조회합니다.

      이번에 반환할 수 있는 최대 금액은
      원승인금액에서 이전 반환금액을 제외하여 계산합니다.

```text
반환 가능금액
= 기존 승인금액 - 기존 누적 반환금액
```
</section>

  <section class="case-study-section" aria-labelledby="journey-title" markdown="1">

## 9. 여정변경 처리 {#journey-title}

여정변경은 기존 거래 반환과
신규 거래 승인이 함께 발생하는 복합 상황입니다.

```text
기존 승차권
│
└── 반환 대상 계산
│
▼
기존 승인 분배결과 기준 반환금액 분배

변경된 승차권
│
└── 신규 수납 및 결제 요청
│
▼
신규 승인금액 분배
```

여정변경에서는 다음 정보를 모두 비교할 수 있어야 합니다.

- 기존 수납정보
- 기존 결제정보
- 기존 승인 분배결과
- 기존 반환내역
- 신규 수납정보
- 신규 결제 요청정보
- 변경 수수료

반환과 신규 승인은 서로 다른 데이터 집합을 사용하지만,
각각 공통 `AllocationContext`로 변환한 후
동일한 분배 메서드를 사용합니다.

```java
public JourneyChangeAllocation allocateJourneyChange(
JourneyChangeRequest request
) {
AllocationContext returnContext =
contextAssembler.assembleReturnPart(request);

AllocationContext approvalContext =
contextAssembler.assembleApprovalPart(request);

List<PaymentAmountMapping> returnMappings =
amountAllocator.allocate(returnContext);

List<PaymentAmountMapping> approvalMappings =
  amountAllocator.allocate(approvalContext);

  return new JourneyChangeAllocation(
  returnMappings,
  approvalMappings
  );
  }
```

계산 메서드를 거래별로 새로 구현하지 않고,
입력 컨텍스트만 반환용과 승인용으로 조립합니다.
</section>

  <section class="case-study-section" aria-labelledby="priority-title" markdown="1">

## 10. 약 14개 결제수단의 우선순위 {#priority-title}

금액은 요청정보에 들어온 순서대로 분배하지 않습니다.

후급, 간편결제, 계좌결제, 카드, 현금 등
약 14개의 결제수단 및 거래 유형마다
업무상 정해진 우선순위가 존재했습니다.

```java
public record RequestedPayment(
PaymentMethod method,
BigDecimal amount,
int allocationPriority
) {
}
```

```java
private List<RequestedPayment> sortByPriority(
List<RequestedPayment> requestedPayments
  ) {
  return requestedPayments.stream()
  .sorted(
  Comparator.comparingInt(
  RequestedPayment::allocationPriority
  )
  )
  .toList();
  }
```

우선순위를 원본 요청의 배열 순서나
서비스 코드의 조건문 순서에 의존하지 않도록 했습니다.

그 결과 요청정보가 어떤 순서로 들어오더라도
동일한 업무 규칙으로 분배할 수 있습니다.
</section>

  <section class="case-study-section" aria-labelledby="allocator-title" markdown="1">

## 11. 하나의 분배 메서드 {#allocator-title}

승인, 반환, 여정변경 모두
동일한 `allocate()` 메서드를 사용합니다.

```java
public List<PaymentAmountMapping> allocate(
AllocationContext context
) {
List<AllocationTarget> targets =
  targetFactory.create(context);

  List<AllocationTarget> orderedTargets =
    sortByPriority(targets);

    AllocationState state =
    AllocationState.from(context);

    List<PaymentAmountMapping> mappings =
      new ArrayList<>();

      for (AllocationTarget target : orderedTargets) {
      if (state.isCompleted()) {
      break;
      }

      AllocatedAmount allocated =
      state.allocateTo(target);

      if (allocated.isZero()) {
      continue;
      }

      mappings.add(
      PaymentAmountMapping.of(
      target.paymentMethod(),
      allocated
      )
      );
      }

      allocationValidator.validate(
      context,
      state,
      mappings
      );

      return List.copyOf(mappings);
      }
```

      거래 유형에 따른 차이는 `targetFactory`가 처리합니다.

```text
승인
요청한 결제금액을 분배 대상으로 생성

반환
기존 승인금액 - 이전 반환금액을
결제수단별 최대 반환 가능금액으로 생성

여정변경
기존 거래는 반환 대상으로,
신규 거래는 승인 대상으로 각각 생성
```

      분배 엔진 자체는 승인인지 반환인지에 따라
      별도의 계산식으로 분기하지 않습니다.
</section>

  <section class="case-study-section" aria-labelledby="return-title" markdown="1">

## 12. 기존 승인금액을 초과하지 않는 반환 {#return-title}

반환에서 가장 중요한 기준은
기존 승인 당시의 결제수단별 분배결과입니다.

예를 들어 승인 당시 다음과 같이 분배되었다면,

```text
카카오페이   10,000원
후급         15,000원
카드          4,000원
현금          1,000원
```

카드로 반환할 수 있는 누적 최대 금액은
4,000원을 초과할 수 없습니다.

기존에 카드에서 1,500원을 반환했다면
현재 반환 가능한 금액은 2,500원입니다.

```java
public BigDecimal returnableAmount(
ApprovedPayment approved,
BigDecimal previouslyReturnedAmount
) {
BigDecimal returnable =
approved.approvedAmount()
.subtract(previouslyReturnedAmount);

if (returnable.signum() < 0) {
throw new InvalidReturnStateException(
approved.method(),
approved.approvedAmount(),
previouslyReturnedAmount
);
}

return returnable;
}
```

이전 승인 분배결과를 기준으로 하지 않고
현재 수납금액을 다시 분배하면,
실제로 승인되지 않은 결제수단에서 반환이 발생할 수 있습니다.

따라서 반환에서는 다음 조건을 항상 검증합니다.

```text
수단별 이번 반환금액
≤ 수단별 승인금액 - 수단별 기존 반환금액
```
</section>

  <section class="case-study-section" aria-labelledby="tax-title" markdown="1">

## 13. 공급가액·부가세·면세금액 분배 {#tax-title}

승인과 반환 요청정보에는
공급가액과 부가세가 포함되어 있지 않았습니다.

따라서 수납정보 또는 기존 승인 분배정보를 기준으로
각 결제 매핑 행의 구성 금액을 계산했습니다.

### 승인

신규 승인에서는 수납정보가 기준입니다.

```text
수납 전체금액
= 수납 공급가액 + 수납 부가세 + 수납 면세금액
```

이를 각 결제수단별 승인금액에 맞게 분배합니다.

### 반환

반환에서는 기존 승인 당시의 분배결과가 기준입니다.

각 결제수단에 승인 당시 귀속된 공급가액과 부가세 범위 안에서
반환 구성 금액을 계산합니다.

```text
수단별 누적 반환 공급가액
≤ 수단별 승인 공급가액

수단별 누적 반환 부가세
≤ 수단별 승인 부가세
```

이를 통해 반환 시점의 계산이
승인 당시 계산과 다른 기준을 사용하는 것을 방지했습니다.
</section>

  <section class="case-study-section" aria-labelledby="fee-title" markdown="1">

## 14. 수수료 분배 {#fee-title}

철도 반환과 여정변경에서는
반환 대상 금액 외에 수수료가 존재할 수 있습니다.

따라서 최종 고객 반환금액과
원거래에서 감소하는 금액을 구분해야 합니다.

```text
원거래 차감금액
= 고객 반환금액 + 반환 수수료
```

수수료 역시 특정 서비스에서 임의로 차감하지 않고,
공통 금액정보의 일부로 포함하여 분배했습니다.

```java
public record FeeAmount(
BigDecimal totalAmount
) {
public FeeAmount {
if (totalAmount == null || totalAmount.signum() < 0) {
throw new InvalidFeeAmountException(totalAmount);
}
}
}
```

분배 결과에는 각 결제수단에 귀속된
반환금액과 수수료가 함께 포함됩니다.

```java
public record PaymentAmountMapping(
PaymentMethod paymentMethod,
BigDecimal paymentAmount,
BigDecimal taxableSupplyAmount,
BigDecimal vatAmount,
BigDecimal taxFreeAmount,
BigDecimal feeAmount
) {
}
```

이를 통해 다음 관계를 확인할 수 있습니다.

```text
결제수단별 원거래 감소금액
= 결제수단별 고객 반환금액
+ 결제수단별 수수료
```
</section>

  <section class="case-study-section" aria-labelledby="state-title" markdown="1">

## 15. 분배 상태 관리 {#state-title}

분배 과정에서는 여러 금액을 동시에 관리해야 합니다.

- 남은 전체 결제금액
- 남은 공급가액
- 남은 부가세
- 남은 면세금액
- 남은 수수료
- 결제수단별 승인 또는 반환 가능금액

이 값을 서비스의 개별 변수로 관리하지 않고
하나의 상태 객체에서 관리했습니다.

```java
public final class AllocationState {

private BigDecimal remainingPaymentAmount;
private BigDecimal remainingSupplyAmount;
private BigDecimal remainingVatAmount;
private BigDecimal remainingTaxFreeAmount;
private BigDecimal remainingFeeAmount;

public AllocatedAmount allocateTo(
AllocationTarget target
) {
BigDecimal paymentAmount =
calculatePaymentAmount(target);

AllocatedAmount allocated =
calculateComponents(
target,
paymentAmount
);

subtract(allocated);
validateNonNegative();

return allocated;
}
}
```

각 분배가 끝날 때마다 남은 금액을 검증하여
중간 계산에서 발생한 오류를 즉시 차단했습니다.
</section>

  <section class="case-study-section" aria-labelledby="validation-title" markdown="1">

## 16. 저장 전 금액 불변식 검증 {#validation-title}

모든 분배가 완료된 뒤 다음 조건을 검증했습니다.

### 요청금액과 매핑금액

```text
결제수단별 매핑금액 합
= 이번 승인 또는 반환 목표금액
```

### 회계 구성 금액

```text
각 행의 공급가액
+ 각 행의 부가세
+ 각 행의 면세금액
= 각 행의 결제금액
```

### 승인 시 전체 수납금액

```text
전체 승인 매핑금액 합
= 수납금액
```

### 반환 가능금액

```text
수단별 누적 반환금액
≤ 수단별 기존 승인금액
```

### 수수료

```text
매핑 수수료 합
= 거래 수수료
```

### 음수 금액 금지

```text
남은 결제금액 ≥ 0
남은 공급가액 ≥ 0
남은 부가세 ≥ 0
남은 면세금액 ≥ 0
남은 수수료 ≥ 0
```

검증에 실패하면 매핑정보를 저장하지 않습니다.

```java
public void validate(
AllocationContext context,
AllocationState state,
List<PaymentAmountMapping> mappings
) {
validatePaymentTotal(context, mappings);
validateTaxComponents(mappings);
validateApprovedLimits(context, mappings);
validateFeeTotal(context, mappings);
state.validateCompleted();
}
```
</section>

  <section class="case-study-section" aria-labelledby="result-title" markdown="1">

## 17. 이 구조로 얻은 효과 {#result-title}

### 승인·반환에 동일한 계산 기준 적용

거래 유형마다 다른 계산 메서드를 만들지 않고
공통 모델과 동일한 분배 엔진을 사용했습니다.

### 복잡한 입력정보를 계산 로직에서 분리

DB 조회와 요청정보 변환은 조립기가 담당하고,
금액 분배 엔진은 정규화된 정보만 사용합니다.

### 기존 승인금액을 기준으로 안전하게 반환

승인 당시의 결제수단별 분배결과를 사용하므로
실제로 승인된 금액보다 많은 반환을 차단할 수 있습니다.

### 여정변경도 동일한 구조로 처리

기존 거래 반환과 신규 거래 승인을
각각 공통 컨텍스트로 변환하여 같은 메서드로 계산합니다.

### 회계 구성 금액의 추적성 확보

결제금액뿐 아니라 공급가액, 부가세,
면세금액과 수수료가 어느 결제수단에 귀속됐는지 확인할 수 있습니다.

### 결제수단 추가 영향 축소

새로운 결제수단이 추가되더라도
요청정보 변환과 우선순위 정의를 추가하면
기존 분배 엔진을 재사용할 수 있습니다.
</section>

  <section class="case-study-section" aria-labelledby="tradeoff-title" markdown="1">

## 18. 트레이드오프 {#tradeoff-title}

### 공통 금액 모델이 복잡해짐

승인, 반환, 여정변경을 모두 표현하기 때문에
단순 승인만 보면 사용하지 않는 정보가 존재할 수 있습니다.

대신 거래별 계산 편차를 제거하고
하나의 검증 체계를 적용할 수 있습니다.

### 정보 조립 과정에서 여러 조회가 필요함

반환과 여정변경은 기존 결제정보,
승인 분배결과와 반환이력을 함께 조회해야 합니다.

조회 대상은 늘어나지만,
잘못된 반환을 방지하려면 반드시 필요한 정보입니다.

### 우선순위 변경이 분배결과에 영향을 줌

약 14개 결제수단의 우선순위는
단순 구현 세부사항이 아니라 업무 규칙입니다.

따라서 우선순위가 변경될 경우
승인과 반환 결과에 미치는 영향을 함께 검증해야 합니다.

### 과거 분배결과 보존이 필요함

현재의 분배 규칙으로 과거 승인거래를 다시 계산하면
당시 결과와 달라질 수 있습니다.

반환에서는 과거 승인 당시 저장된 실제 분배결과를
정합성의 기준으로 사용해야 합니다.
</section>

  <section class="case-study-section" aria-labelledby="retrospective-title" markdown="1">

## 19. 회고 {#retrospective-title}

이 작업의 어려움은 금액을 나누는 공식 자체보다
서로 다른 시점과 목적을 가진 정보를
하나의 기준으로 연결하는 데 있었습니다.

승인에서는 신규 수납정보와 요청정보만 존재하지만,
반환에서는 기존 결제정보와 승인 당시의 분배결과가 필요합니다.

여정변경에서는 기존 거래의 반환정보와
신규 거래의 승인정보를 동시에 확인해야 하며,
수수료까지 함께 반영해야 합니다.

각 상황마다 별도의 계산 로직을 작성하면
같은 금액도 승인, 반환, 여정변경에서
서로 다른 방식으로 해석될 가능성이 있었습니다.

이를 방지하기 위해 다음과 같이 책임을 분리했습니다.

```text
거래 유형 판별과 DB 조회
→ AllocationContextAssembler

요청 및 DB 정보의 공통 모델 변환
→ AllocationContext

결제수단별 우선순위와 금액 분배
→ AmountAllocator

승인한도·세금·수수료·합계 검증
→ AllocationValidator
```

그 결과 승인과 반환 여부와 관계없이
동일한 메서드에서 금액을 분배할 수 있게 되었고,
기존 승인금액을 초과한 반환이나
수납정보와 결제정보의 불일치를
DB 저장 이전에 차단할 수 있었습니다.

복잡한 금액 계산에서는 공식을 짧게 만드는 것보다,

- 어떤 데이터를 기준으로 계산했는지
- 승인과 반환이 동일한 규칙을 사용하는지
- 기존 승인금액을 초과하지 않는지
- 공급가액과 부가세, 수수료가 어디에 귀속되는지
- 최종 결과가 수납정보와 일치하는지

를 코드로 명확하게 설명할 수 있는 구조가 더 중요하다고 생각합니다.
</section>

</article>
---
layout: default
title: 결제 수단 라우팅과 실패 복구
description: 결제 수단별 구현체를 동적으로 선택하는 실행 구조와 망취소·배치 취소를 이용한 실패 복구 설계 사례입니다.
body_class: case-study-page
image: /assets/images/payment-recovery.svg
---

<article class="case-study">

    <header class="case-study-header">
        <p class="case-study-header__eyebrow">
            Architecture Case Study 01
        </p>

        <h1 class="case-study-header__title">
            결제 수단을 직접 호출하지 않고<br>
            하나의 실행 흐름으로 관리하기
        </h1>

        <p class="case-study-header__description">
            VAN, 간편결제, 계좌이체 등 결제 수단마다 연동 규격과
            승인·취소 방식은 달랐지만, 이를 사용하는 비즈니스 서비스가
            각 구현체를 직접 선택하고 호출해서는 안 된다고 판단했습니다.
        </p>

        <p class="case-study-header__description">
            결제 인터페이스, 구현체 저장기, 실행기를 분리하여
            결제 수단이 추가되어도 기존 호출 흐름을 수정하지 않도록 구성했습니다.
            또한 승인 이후 발생하는 실패를 망취소와 취소 배치로 복구할 수 있도록 설계했습니다.
        </p>

        <div class="case-study-header__tags" aria-label="관련 주제">
            <span>Payment Routing</span>
            <span>Strategy Registry</span>
            <span>Net Cancel</span>
            <span>Recovery Batch</span>
        </div>
    </header>

    <section class="case-study-summary" aria-labelledby="summary-title">

        ## 한눈에 보기 {#summary-title}

        <div class="case-study-summary__grid">
            <div class="case-study-summary__item">
                <strong>문제</strong>
                <p>
                    결제 수단이 늘어날수록 호출 서비스에 조건문과
                    수단별 의존성이 계속 추가될 수 있었습니다.
                </p>
            </div>

            <div class="case-study-summary__item">
                <strong>설계</strong>
                <p>
                    결제 구현체를 애플리케이션 시작 시점에 key 기반 Map으로 등록하고,
                    실행기가 요청에 맞는 구현체를 찾아 실행하도록 구성했습니다.
                </p>
            </div>

            <div class="case-study-summary__item">
                <strong>복구</strong>
                <p>
                    망취소와 일반 취소를 구분하고,
                    실패한 취소는 유형별로 DB에 적재하여 배치가 재처리하도록 했습니다.
                </p>
            </div>
        </div>

    </section>

    <section class="case-study-section" aria-labelledby="background-title">

        ## 1. 배경 {#background-title}

        통합 결제 시스템은 여러 결제 수단과 외부 결제망을 지원해야 했습니다.

        결제 수단마다 다음 항목이 달랐습니다.

        - 외부 기관에 전달해야 하는 요청 정보
        - 승인 요청 방식
        - 취소 요청 방식
        - 망취소 지원 여부
        - 외부 거래 식별 방식
        - 응답 코드와 성공 판단 기준

        각 비즈니스 서비스가 결제 수단을 직접 판단하는 구조에서는
        새로운 수단이 추가될 때마다 호출부를 수정해야 합니다.

        ```java
        if (paymentType == CARD) {
        cardPaymentService.approve(command);
        } else if (paymentType == EASY_PAY) {
        easyPayService.approve(command);
        } else if (paymentType == ACCOUNT) {
        accountPaymentService.approve(command);
        }
        ```

        이 방식은 처음에는 단순하지만 결제 수단과 행위가 늘어날수록
        조건문이 여러 서비스로 퍼집니다.

        또한 승인, 롤백, 취소 처리 방식이 호출부에 노출되기 때문에
        결제 수단별 정책을 한곳에서 관리하기 어려워집니다.

    </section>

    <section class="case-study-section" aria-labelledby="goal-title">

        ## 2. 설계 목표 {#goal-title}

        다음 기준을 세웠습니다.

        ### 호출 서비스는 구체적인 결제 수단을 알지 않는다

        호출하는 쪽에서는 결제 수단의 구현체를 직접 주입받지 않습니다.

        결제에 필요한 정보와 실행할 행위만 Command와 Detail에 담아
        공통 실행기에 전달합니다.

        ### 새로운 결제 수단이 기존 흐름을 변경하지 않는다

        결제 인터페이스를 구현하고 식별 key를 제공하면
        애플리케이션 시작 시점에 자동으로 등록되어야 합니다.

        ### 결제 수단별 차이는 구현체 내부에 둔다

        승인, 롤백, 취소의 실제 처리 방식은 각 결제 수단 구현체가 책임집니다.

        ### 실패 이후 처리도 결제 수단의 정책에 포함한다

        망취소가 필요한지, 일반 취소로 처리할지,
        실패한 취소를 어떤 유형으로 재처리할지를 구현체가 결정할 수 있어야 합니다.

    </section>

    <section class="case-study-section" aria-labelledby="architecture-title">

        ## 3. 전체 구조 {#architecture-title}

        ![결제 라우팅과 실패 복구 구조]({{ '/assets/images/payment-recovery.svg' | relative_url }})

        전체 구조는 네 가지 역할로 구분했습니다.

        ```text
        PaymentCommand / PaymentDetail
        │
        ▼
        PaymentExecutor
        │
        ▼
        PaymentRegistry
        key → implementation
        │
        ▼
        PaymentProcessor
        │
        ┌───────┼────────┐
        ▼       ▼        ▼
        APPROVE  ROLLBACK  CANCEL
        ```

        ### PaymentCommand

        어떤 행위를 실행할지 표현합니다.

        ```text
        APPROVE
        ROLLBACK
        CANCEL
        ```

        ### PaymentDetail

        실제 결제를 수행하기 위해 필요한 정보를 담습니다.

        - 결제 수단 식별 key
        - 거래 식별자
        - 결제 금액
        - 원승인 정보
        - 결제 수단별 추가 정보

        ### PaymentRegistry

        애플리케이션 시작 시점에 결제 인터페이스 구현체를 수집하고,
        각 구현체가 제공하는 key를 기준으로 Map에 저장합니다.

        ### PaymentExecutor

        Command에 포함된 결제 수단 key로 구현체를 조회하고,
        해당 구현체의 `process()`를 실행합니다.

        호출 서비스는 어떤 클래스가 선택되는지 알 필요가 없습니다.

    </section>

    <section class="case-study-section" aria-labelledby="interface-title">

        ## 4. 공통 결제 인터페이스 {#interface-title}

        아래 코드는 실제 업무 구조를 설명하기 위해
        이름과 세부 구현을 일반화한 예시입니다.

        ```java
        public interface PaymentProcessor {

        String key();

        PaymentResult process(
        PaymentCommand command,
        PaymentDetail detail
        );
        }
        ```

        모든 결제 수단은 두 가지 정보를 제공합니다.

        1. 자신을 식별할 수 있는 `key`
        2. 결제 행위를 처리하는 `process()`

        예를 들어 카드 결제와 간편결제는 같은 인터페이스를 구현하지만
        내부 승인 규격과 취소 방식은 서로 다르게 가질 수 있습니다.

        ```java
        @Component
        public class CardPaymentProcessor implements PaymentProcessor {

        @Override
        public String key() {
        return "CARD";
        }

        @Override
        public PaymentResult process(
        PaymentCommand command,
        PaymentDetail detail
        ) {
        return switch (command.action()) {
        case APPROVE -> approve(command, detail);
        case ROLLBACK -> rollback(command, detail);
        case CANCEL -> cancel(command, detail);
        };
        }
        }
        ```

        `process()`는 전달받은 Command의 행위를 판단하고
        해당 결제 수단의 승인, 롤백 또는 취소 메서드를 실행합니다.

    </section>

    <section class="case-study-section" aria-labelledby="registry-title">

        ## 5. 구현체 저장기 {#registry-title}

        Spring이 애플리케이션을 시작하면서
        `PaymentProcessor`를 구현한 Bean을 모두 주입합니다.

        저장기는 각 구현체의 key와 객체를 Map으로 구성합니다.

        ```java
        @Component
        public class PaymentRegistry {

        private final Map<String, PaymentProcessor> processors;

        public PaymentRegistry(List<PaymentProcessor> processors) {
        this.processors = processors.stream()
        .collect(Collectors.toUnmodifiableMap(
        PaymentProcessor::key,
        Function.identity()
        ));
        }

        public PaymentProcessor getRequired(String key) {
        PaymentProcessor processor = processors.get(key);

        if (processor == null) {
        throw new UnsupportedPaymentMethodException(key);
        }

        return processor;
        }
        }
        ```

        새로운 결제 수단을 추가할 때는 다음 구현만 추가하면 됩니다.

        ```java
        @Component
        public class NewPaymentProcessor implements PaymentProcessor {

        @Override
        public String key() {
        return "NEW_PAYMENT";
        }

        @Override
        public PaymentResult process(
        PaymentCommand command,
        PaymentDetail detail
        ) {
        return switch (command.action()) {
        case APPROVE -> approve(command, detail);
        case ROLLBACK -> rollback(command, detail);
        case CANCEL -> cancel(command, detail);
        };
        }
        }
        ```

        기존 실행기나 호출 서비스를 수정하지 않아도
        새 구현체가 Registry에 포함됩니다.

    </section>

    <section class="case-study-section" aria-labelledby="executor-title">

        ## 6. 공통 실행기 {#executor-title}

        실행기는 결제 수단별 비즈니스 규칙을 갖지 않습니다.

        Command와 Detail에서 key를 확인하고,
        Registry에서 구현체를 찾아 실행하는 역할만 수행합니다.

        ```java
        @Component
        public class PaymentExecutor {

        private final PaymentRegistry paymentRegistry;

        public PaymentExecutor(PaymentRegistry paymentRegistry) {
        this.paymentRegistry = paymentRegistry;
        }

        public PaymentResult execute(
        PaymentCommand command,
        PaymentDetail detail
        ) {
        PaymentProcessor processor =
        paymentRegistry.getRequired(detail.paymentKey());

        return processor.process(command, detail);
        }
        }
        ```

        비즈니스 서비스에서는 구체적인 결제 서비스를 호출하지 않습니다.

        ```java
        public PaymentResult approve(PaymentRequest request) {
        PaymentCommand command =
        PaymentCommand.approve(request.transactionId());

        PaymentDetail detail =
        PaymentDetail.from(request);

        return paymentExecutor.execute(command, detail);
        }
        ```

        호출부에서는 다음 흐름만 보입니다.

        ```text
        결제 정보 생성
        → 실행 행위 지정
        → 공통 실행기 호출
        ```

        결제 수단별 분기와 외부 기관 규격은 호출부에 노출되지 않습니다.

    </section>

    <section class="case-study-section" aria-labelledby="approval-title">

        ## 7. 승인 과정의 실패 처리 {#approval-title}

        승인 처리 도중 예외가 발생하면
        거래 상태와 결제 수단의 특성에 따라 후속 처리를 결정합니다.

        ```java
        private PaymentResult approve(
        PaymentCommand command,
        PaymentDetail detail
        ) {
        try {
        return executeApproval(command, detail);
        } catch (RuntimeException approvalFailure) {
        recoverApprovalFailure(
        command,
        detail,
        approvalFailure
        );

        throw approvalFailure;
        }
        }
        ```

        복구 방식은 크게 두 가지입니다.

        ### 망취소가 필요한 거래

        외부 승인 직후 내부 처리에서 문제가 발생했고,
        해당 결제망이 망취소를 지원하는 거래라면 즉시 망취소를 시도합니다.

        ```java
        private void recoverApprovalFailure(
        PaymentCommand command,
        PaymentDetail detail,
        RuntimeException cause
        ) {
        if (detail.requiresNetCancel()) {
        executeNetCancelOrRegister(
        command,
        detail,
        cause
        );
        return;
        }

        cancellationService.registerCancel(
        command,
        detail,
        cause
        );
        }
        ```

        ### 일반 취소가 필요한 거래

        망취소 대상이 아니라면
        외부 취소를 바로 실행하지 않고 일반 취소 대상으로 등록합니다.

        일반 결제 취소의 기본 처리 주체는 배치입니다.

    </section>

    <section class="case-study-section" aria-labelledby="net-cancel-title">

        ## 8. 망취소 실패 처리 {#net-cancel-title}

        망취소는 승인 직후 즉시 정합성을 회복하기 위한 처리입니다.

        그러나 최초 승인 흐름에서 이미 네트워크나 외부 기관 문제가 발생했다면
        망취소 역시 실패할 가능성이 높습니다.

        ```java
        private void executeNetCancelOrRegister(
        PaymentCommand command,
        PaymentDetail detail,
        RuntimeException approvalFailure
        ) {
        try {
        executeNetCancel(command, detail);
        } catch (RuntimeException netCancelFailure) {
        cancellationRepository.save(
        CancellationTask.netCancel(
        command.transactionId(),
        detail,
        approvalFailure,
        netCancelFailure
        )
        );
        }
        }
        ```

        망취소가 실패하면 단순한 취소 실패로 저장하지 않습니다.

        ```text
        type = NET_CANCEL
        ```

        유형을 명확하게 구분하여 취소 배치에 등록합니다.

        이를 통해 배치에서는 해당 거래가 일반 사용자 취소인지,
        승인 실패 이후 정합성을 복구하기 위한 망취소인지 판단할 수 있습니다.

    </section>

    <section class="case-study-section" aria-labelledby="cancel-title">

        ## 9. 일반 결제 취소는 배치로 처리 {#cancel-title}

        일반 결제 취소는 기본적으로 요청 시점에
        외부 취소 API를 즉시 호출하지 않습니다.

        ```text
        취소 요청
        ↓
        취소 대상 DB 등록
        ↓
        취소 배치 조회
        ↓
        PaymentExecutor 실행
        ↓
        해당 결제 수단 process(CANCEL)
        ↓
        외부 결제 취소
        ```

        취소 요청을 DB에 먼저 적재한 뒤
        배치가 처리하도록 한 이유는 다음과 같습니다.

        - 외부 결제망 장애가 사용자 요청까지 전파되는 것을 줄임
        - 실패한 취소를 동일한 기준으로 다시 처리할 수 있음
        - 취소 대상과 처리 상태를 DB에서 추적할 수 있음
        - 결제망별 장애가 발생해도 복구 대상이 유실되지 않음
        - 일시적인 장애와 영구적인 실패를 구분할 수 있음

        일반 취소는 다음 유형으로 저장됩니다.

        ```text
        type = CANCEL
        ```

        망취소 대상과 일반 취소 대상을 같은 테이블에서 관리하더라도
        유형을 통해 처리 목적과 재처리 방식을 구분할 수 있습니다.

    </section>

    <section class="case-study-section" aria-labelledby="batch-title">

        ## 10. 취소 배치 실행 {#batch-title}

        배치는 처리되지 않은 취소 대상을 조회하고,
        기존 결제 실행기와 동일한 흐름을 사용합니다.

        ```java
        public void process(CancellationTask task) {
        PaymentCommand command =
        PaymentCommand.cancel(task.transactionId());

        PaymentDetail detail =
        task.toPaymentDetail();

        try {
        paymentExecutor.execute(command, detail);
        cancellationRepository.complete(task.id());
        } catch (RuntimeException cancelFailure) {
        cancellationRepository.recordFailure(
        task.id(),
        cancelFailure.getMessage()
        );
        }
        }
        ```

        배치가 직접 특정 결제 수단의 서비스를 호출하지 않는 것이 중요합니다.

        ```text
        CancellationBatch
        │
        ▼
        PaymentExecutor
        │
        ▼
        PaymentRegistry
        │
        ▼
        PaymentProcessor.process(CANCEL)
        ```

        승인과 취소가 동일한 라우팅 구조를 사용하므로
        결제 수단이 추가되어도 배치 코드를 수정할 필요가 없습니다.

    </section>

    <section class="case-study-section" aria-labelledby="mode-title">

        ## 11. 배치와 즉시 취소 전환 {#mode-title}

        일반 취소의 기본 방식은 배치 처리입니다.

        하지만 취소 배치 자체에 장애가 발생하면
        새로운 취소 요청이 계속 대기 상태로 누적될 수 있습니다.

        이 상황에 대응하기 위해 DB 설정값으로
        취소 실행 방식을 전환할 수 있도록 구성했습니다.

        ```text
        BATCH
        취소 요청을 DB에 적재
        배치가 외부 취소 실행

        IMMEDIATE
        서버가 외부 취소를 즉시 실행
        ```

        개념적으로는 다음과 같은 흐름입니다.

        ```java
        public void requestCancellation(
        PaymentCommand command,
        PaymentDetail detail
        ) {
        CancellationMode mode =
        cancellationPolicy.currentMode();

        if (mode == CancellationMode.BATCH) {
        cancellationRepository.save(
        CancellationTask.cancel(command, detail)
        );
        return;
        }

        executeImmediatelyOrRegister(command, detail);
        }
        ```

        즉시 취소 모드에서도 실패 가능성을 제거할 수는 없습니다.

        ```java
        private void executeImmediatelyOrRegister(
        PaymentCommand command,
        PaymentDetail detail
        ) {
        try {
        paymentExecutor.execute(
        command.toCancel(),
        detail
        );
        } catch (RuntimeException immediateFailure) {
        cancellationRepository.save(
        CancellationTask.cancel(
        command,
        detail,
        immediateFailure
        )
        );
        }
        }
        ```

        즉시 취소가 실패하더라도 거래를 유실하지 않고
        다시 배치 처리할 수 있도록 DB에 적재합니다.

    </section>

    <section class="case-study-section" aria-labelledby="flow-title">

        ## 12. 전체 실패 복구 흐름 {#flow-title}

        ### 승인 중 망취소 대상 예외

        ```text
        APPROVE
        │
        ├── 성공
        │     └── 승인 완료
        │
        └── 예외 발생
        │
        └── 망취소 필요
        │
        ├── 망취소 성공
        │     └── 복구 완료
        │
        └── 망취소 실패
        └── NET_CANCEL 유형으로 DB 등록
        ```

        ### 승인 중 일반 취소 대상 예외

        ```text
        APPROVE
        │
        └── 예외 발생
        │
        └── 일반 취소 필요
        └── CANCEL 유형으로 DB 등록
        ```

        ### 일반 사용자 취소

        ```text
        취소 요청
        │
        ├── BATCH 모드
        │     └── CANCEL 유형으로 DB 등록
        │
        └── IMMEDIATE 모드
        │
        ├── 즉시 취소 성공
        │     └── 취소 완료
        │
        └── 즉시 취소 실패
        └── CANCEL 유형으로 DB 등록
        ```

    </section>

    <section class="case-study-section" aria-labelledby="failure-table-title">

        ## 13. 실패 상황별 처리 {#failure-table-title}

        | 상황 | 처리 방식 |
        |---|---|
        | 승인 정상 완료 | 승인 결과 반환 |
        | 승인 과정에서 망취소 필요 예외 발생 | 즉시 망취소 시도 |
        | 망취소 성공 | 복구 완료 |
        | 망취소 실패 | `NET_CANCEL` 유형으로 취소 배치 등록 |
        | 일반 취소 필요 | `CANCEL` 유형으로 취소 배치 등록 |
        | 일반 취소 배치 성공 | 취소 완료 |
        | 일반 취소 배치 실패 | DB 상태와 실패 이력 유지 |
        | 배치 장애 중 신규 취소 요청 | DB 설정을 통해 즉시 취소 모드 사용 가능 |
        | 즉시 취소 성공 | 취소 완료 |
        | 즉시 취소 실패 | `CANCEL` 유형으로 DB 재등록 |

        복구 과정의 어떤 단계에서 실패하더라도
        취소 대상이 메모리나 로그에만 남지 않도록 했습니다.

        최종적으로 처리되지 않은 거래는 DB에 남아
        다음 배치 또는 운영 조치의 대상이 됩니다.

    </section>

    <section class="case-study-section" aria-labelledby="result-title">

        ## 14. 이 구조로 얻은 효과 {#result-title}

        ### 결제 수단 추가가 단순해짐

        새로운 결제 수단은 `PaymentProcessor`를 구현하고
        고유한 key를 제공하면 기존 실행 흐름에 참여합니다.

        호출 서비스, 실행기, 배치를 수정할 필요가 없습니다.

        ### 호출부의 결제 수단 분기 제거

        호출부는 결제 수단 구현체를 직접 선택하지 않습니다.

        Command와 Detail을 구성하고 실행기를 호출하는
        동일한 형태로 모든 결제를 처리합니다.

        ### 승인과 취소가 같은 실행 구조를 사용

        사용자 요청, 실패 복구, 취소 배치가
        모두 `PaymentExecutor`를 통해 결제 구현체를 호출합니다.

        동일한 결제 수단 선택 규칙을 여러 곳에서 중복 구현하지 않습니다.

        ### 결제 수단별 정책을 한곳에서 관리

        승인, 롤백, 취소와 실패 복구 정책이
        각 결제 수단 구현체 내부에 모입니다.

        외부 결제망의 특수 규격이 상위 비즈니스 서비스로 퍼지는 것을 줄였습니다.

        ### 취소 대상의 유실 방지

        망취소 실패와 일반 취소를 유형별로 DB에 적재하여
        복구되지 않은 거래를 추적하고 재처리할 수 있습니다.

    </section>

    <section class="case-study-section" aria-labelledby="tradeoff-title">

        ## 15. 트레이드오프 {#tradeoff-title}

        ### 구현체 내부의 책임이 커질 수 있음

        각 결제 수단 구현체가 승인, 롤백, 취소를 모두 처리하므로
        구현체의 크기가 커질 수 있습니다.

        이를 줄이기 위해 공통 실행 흐름은 유지하되,
        전문 생성, 외부 통신, 응답 변환 등의 세부 책임은
        필요에 따라 내부 컴포넌트로 분리할 수 있습니다.

        ### key 관리가 중요해짐

        Registry가 문자열 key를 기준으로 구현체를 선택하므로
        잘못된 key나 중복 key는 실행 오류로 이어질 수 있습니다.

        따라서 애플리케이션 시작 시점에 중복 key를 검증하고,
        지원하지 않는 key는 실행 전에 명확한 예외로 차단해야 합니다.

        ### 배치 취소는 즉시 완료되지 않음

        취소 요청과 실제 외부 취소 사이에 시간 차이가 생깁니다.

        대신 외부 장애를 사용자 요청과 분리하고,
        실패한 취소를 안정적으로 재처리할 수 있습니다.

        ### DB 설정에 따른 실행 모드 관리가 필요함

        배치와 즉시 취소를 전환할 수 있는 대신,
        현재 어떤 모드로 운영 중인지 명확하게 관리해야 합니다.

        모드 전환은 장애 대응을 위한 운영 수단이며,
        평상시 기본 흐름은 배치 취소로 유지합니다.

    </section>

    <section class="case-study-section" aria-labelledby="retrospective-title">

        ## 16. 회고 {#retrospective-title}

        이 설계에서 가장 중요했던 부분은
        공통화를 위해 결제 수단의 차이를 없애는 것이 아니었습니다.

        호출하는 쪽에서는 동일한 방식으로 사용할 수 있게 만들되,
        각 결제 수단의 승인, 롤백, 취소 정책은 구현체 내부에서
        명확하게 표현할 수 있도록 경계를 설정했습니다.

        또한 예외 처리에서 보상 작업을 한 번 시도하는 것으로 끝내지 않았습니다.

        망취소와 일반 취소를 구분하고,
        즉시 처리가 실패하면 DB에 복구 대상을 남겨
        배치가 다시 처리할 수 있도록 했습니다.

        이를 통해 결제 코드는 간결해졌지만,
        단순히 코드 줄 수만 줄어든 것은 아닙니다.

        - 결제 수단 선택 책임은 Registry로 이동했습니다.
        - 실행 책임은 Executor로 이동했습니다.
        - 수단별 정책은 PaymentProcessor 구현체에 남겼습니다.
        - 실패한 취소의 복구 책임은 DB와 Batch로 분리했습니다.

        각 책임의 위치가 명확해지면서
        새로운 결제 수단 추가와 장애 대응 모두
        기존보다 예측 가능한 구조가 되었습니다.

    </section>

</article>
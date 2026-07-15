---
layout: default
title: 복호화 실패 원인 역추적과 공식 가이드 개선
description: 공식 연동 가이드와 실제 승인 데이터의 암호화 규격이 다른 문제를 테스트 기반으로 분석하고, 실제 규격을 확인하여 외부 기관의 공식 가이드 수정을 이끌어낸 사례입니다.
body_class: case-study-page
---

<article class="case-study">

  <header class="case-study-header">
    <p class="case-study-header__eyebrow">
      Troubleshooting Case Study 03
    </p>

    <h1 class="case-study-header__title">
      공식 문서로 해결되지 않던 복호화 실패를<br>
      테스트 가능한 가설로 역추적하기
    </h1>

    <p class="case-study-header__description">
      간편결제 승인 결과를 복호화하는 과정에서
      공식 가이드의 Java 예제를 그대로 구현했지만
      모든 승인 응답에서 패딩 오류가 발생했습니다.
    </p>

    <p class="case-study-header__description">
      개발 일정상 가이드가 수정될 때까지 기다릴 수 없는 상황이었기 때문에,
      입력 데이터 형식, RSA 복호화 결과 처리 방식,
      AES Key 길이와 IV 구성 방식을 각각 독립적인 변수로 나누어 검증했습니다.
      최종적으로 문서와 실제 적용 규격이 다르다는 점을 확인했고,
      분석 결과는 외부 기관의 공식 가이드 수정으로 이어졌습니다.
    </p>

    <div class="case-study-header__tags" aria-label="관련 주제">
      <span>Encryption Integration</span>
      <span>Hypothesis Testing</span>
      <span>Known Plaintext</span>
      <span>External API</span>
      <span>Technical Communication</span>
    </div>
  </header>

<section class="case-study-summary" aria-labelledby="summary-title" markdown="1">

## 한눈에 보기 {#summary-title}

<div class="case-study-summary__grid">
  <div class="case-study-summary__item">
    <strong>문제</strong>
    <p>
      공식 가이드의 AES-128 예제로 승인 결과를 복호화하면
      지속적으로 패딩 오류가 발생하여 결제 연동을 진행할 수 없었습니다.
    </p>
  </div>

  <div class="case-study-summary__item">
    <strong>접근</strong>
    <p>
      암호문 인코딩, RSA 결과 처리, AES Key 길이,
      IV 생성 방식을 각각 분리하여 테스트 조합을 구성했습니다.
    </p>
  </div>

  <div class="case-study-summary__item">
    <strong>결과</strong>
    <p>
      실제 데이터가 문서와 다른 AES-256 기반 규격과
      별도의 고정 IV를 사용한다는 점을 확인했고,
      누락 필드와 함께 공식 가이드 수정에 반영되었습니다.
    </p>
  </div>
</div>
</section>

<section class="case-study-section" aria-labelledby="background-title" markdown="1">

## 1. 배경 {#background-title}

통합 결제 시스템에 간편결제 승인 기능을 연동하면서,
승인 응답에 포함된 암호화 데이터를 복호화해야 했습니다.

외부 기관에서 제공한 가이드에는 다음과 같은 Java 예제가 포함되어 있었습니다.

```text
암호문 입력 형식
→ HEX 문자열

RSA 복호화 결과
→ 문자열 변환
→ HEX 디코딩
→ 16바이트 AES Key

AES 알고리즘
→ AES-128-CBC

IV
→ AES Key와 동일한 값
```

가이드를 기준으로 구현했지만,
실제 승인 응답을 복호화하면 다음 예외가 반복적으로 발생했습니다.

```text
javax.crypto.BadPaddingException
```

패딩 오류는 단순히 마지막 패딩 설정만 잘못됐다는 의미가 아닐 수 있습니다.

다음 중 하나라도 다르면 동일한 오류가 발생할 수 있습니다.

- 암호문 인코딩 방식
- 암호화 Key
- Key 길이
- IV
- CBC 블록 구성
- RSA 복호화 결과의 처리 방식
- 암호문 데이터 손상

공식 문서를 그대로 구현했는데도 실패했기 때문에
애플리케이션 코드만 반복해서 수정하는 방식으로는
원인을 찾기 어려운 상황이었습니다.
</section>

<section class="case-study-section" aria-labelledby="constraints-title" markdown="1">

## 2. 제약 조건 {#constraints-title}

### 개발 일정이 촉박함

외부 기관에서 가이드를 다시 확인하고 수정할 때까지
연동 개발을 중단할 수 없었습니다.

승인 연동이 완료되지 않으면
후속 결제 시나리오와 통합 테스트도 진행할 수 없었습니다.

### 실제 암호화 구현을 직접 확인할 수 없음

외부 기관의 서버 코드나 암호화 내부 구현에는 접근할 수 없었습니다.

확인할 수 있는 정보는 다음 정도였습니다.

- 공식 가이드
- 암호화된 승인 응답
- RSA로 복호화할 수 있는 Key 데이터
- 복호화 성공 시 예상되는 평문 구조
- 카드사별 테스트 승인 결과

### 무작위 변경으로는 원인을 찾기 어려움

암호화 연동에는 여러 변수가 동시에 존재합니다.

```text
입력 인코딩
× RSA 결과 처리
× AES Key 길이
× IV 생성 방식
× Padding 방식
```

변수를 한 번에 여러 개 변경하면
우연히 일부 문자열이 출력되더라도
어떤 조건이 정확한 원인이었는지 증명하기 어렵습니다.

따라서 각 조건을 독립적인 가설로 나누고
재현 가능한 테스트를 만들어야 했습니다.
</section>

<section class="case-study-section" aria-labelledby="symptom-title" markdown="1">

## 3. 최초 증상 {#symptom-title}

공식 예제를 적용한 초기 흐름은 다음과 같았습니다.

```text
승인 응답의 암호문
        │
        ▼
HEX 문자열 디코딩
        │
        ▼
RSA 복호화 결과를 문자열로 변환
        │
        ▼
문자열을 다시 HEX 디코딩
        │
        ▼
앞 16바이트를 AES Key로 사용
        │
        ▼
AES-128-CBC 복호화
IV = AES Key
        │
        ▼
BadPaddingException
```

모든 테스트 데이터에서 동일하게 실패했기 때문에
단순한 특정 카드사 데이터 문제로 보기 어려웠습니다.

또한 승인 자체는 외부 기관에서 정상적으로 처리되고 있었으므로,
문제 범위는 승인 응답의 암호화 데이터 처리 구간으로 좁힐 수 있었습니다.
</section>

<section class="case-study-section" aria-labelledby="hypothesis-title" markdown="1">

## 4. 가능한 원인을 가설로 분리 {#hypothesis-title}

한 번에 하나의 조건만 변경할 수 있도록
복호화 과정을 다음 변수로 분리했습니다.

### 암호문 입력 형식

```text
HEX
Base64
Raw Byte
```

### RSA 복호화 결과 처리

```text
문자열 변환 후 HEX 디코딩
Raw Byte 그대로 사용
앞 16바이트만 사용
전체 32바이트 사용
```

### AES Key 길이

```text
AES-128
AES-256
```

### IV 후보

```text
Key와 동일한 값
Key의 앞 16바이트
암호문의 앞 16바이트
0으로 채운 IV
별도의 고정 IV
```

### Padding과 Mode

```text
AES/CBC/PKCS5Padding
AES/CBC/NoPadding
```

가설을 분리한 목적은
가능한 조합을 무작정 시도하는 것이 아니었습니다.

각 실행 결과를 비교하여
어떤 단계까지는 정상인지 판단할 수 있도록 만드는 것이 목적이었습니다.
</section>

<section class="case-study-section" aria-labelledby="test-harness-title" markdown="1">

## 5. 조합을 반복 검증할 수 있는 테스트 구조 {#test-harness-title}

복호화 코드를 직접 수정하면서 매번 승인 테스트를 수행하면
실험 속도가 느리고 결과 비교도 어렵습니다.

따라서 암호화 조건을 파라미터로 바꾸어
같은 데이터를 반복 검증할 수 있는 테스트 구조를 만들었습니다.

아래 코드는 실제 구조를 설명하기 위해
세부 명칭과 보안값을 일반화한 예시입니다.

```java
record DecryptionCandidate(
    CipherTextEncoding cipherTextEncoding,
    RsaResultMode rsaResultMode,
    AesKeySize aesKeySize,
    IvStrategy ivStrategy,
    String transformation
    ) {
}
```

```java
record DecryptionAttempt(
    DecryptionCandidate candidate,
    boolean decrypted,
    boolean validStructure,
    String preview,
    String failureReason
    ) {
}
```

각 후보 조합을 동일한 입력 데이터에 적용했습니다.

```java
List<DecryptionAttempt> verify(
    EncryptedApprovalData encryptedData,
    byte[] rsaDecryptedKey
    ) {
    List<DecryptionAttempt> results = new ArrayList<>();

    for (DecryptionCandidate candidate : candidates()) {
        try {
            byte[] cipherText = decodeCipherText(
                encryptedData.value(),
                candidate.cipherTextEncoding()
                );

            SecretKey key = createAesKey(
                rsaDecryptedKey,
                candidate.rsaResultMode(),
                candidate.aesKeySize()
                );

            IvParameterSpec iv = createIv(
                candidate.ivStrategy(),
                key,
                cipherText
                );

            String plainText = decrypt(
                cipherText,
                key,
                iv,
                candidate.transformation()
                );

            results.add(
                DecryptionAttempt.success(
                candidate,
                plainText,
                approvalPlainTextValidator.isValid(plainText)
                )
                );
        } catch (RuntimeException exception) {
        results.add(
            DecryptionAttempt.failure(
            candidate,
            exception.getClass().getSimpleName()
            )
            );
    }
}

return List.copyOf(results);
}
```

이 구조를 통해 다음 항목을 동일한 기준으로 비교할 수 있었습니다.

- 복호화 예외 발생 여부
- 출력 문자열의 길이
- 출력 가능한 문자 비율
- 예상 필드 존재 여부
- 평문 구조의 유효성
- 카드사별 반복 성공 여부
</section>

<section class="case-study-section" aria-labelledby="validation-title" markdown="1">

## 6. 단순히 예외가 없는 결과를 성공으로 보지 않음 {#validation-title}

잘못된 Key나 IV를 사용해도
우연히 패딩 검증을 통과할 가능성을 완전히 배제할 수는 없습니다.

따라서 예외가 발생하지 않는 것만으로
복호화 성공이라고 판단하지 않았습니다.

승인 평문에는 일정한 구조와 식별 가능한 필드가 있었습니다.
```text
field_name=value
field_name=value
...
```

특히 정상 평문에 존재해야 하는
안정적인 필드 구분자와 카드정보 필드를 기준점으로 사용했습니다.

```java
public final class ApprovalPlainTextValidator {

    public boolean isValid(String plainText) {
        if (plainText == null || plainText.isBlank()) {
            return false;
        }

        return plainText.contains("known_field=")
            && hasValidDelimiterStructure(plainText)
            && hasExpectedFieldCount(plainText)
            && containsValidCardDataFormat(plainText);
    }
}
```

검증 기준은 다음과 같았습니다.

```text
복호화 예외가 발생하지 않는가
AND
예상 필드명이 존재하는가
AND
필드 구분 구조가 유효한가
AND
카드정보 형식이 정상적인가
AND
여러 테스트 데이터에서 동일하게 재현되는가
```

즉, 읽을 수 있는 문자열이 일부 출력됐다는 이유만으로
정답이라고 결론 내리지 않았습니다.
</section>

<section class="case-study-section" aria-labelledby="partial-title" markdown="1">

## 7. 부분 성공 결과를 이용해 원인을 좁힘 {#partial-title}

일부 조합에서는 전체 복호화에 성공하지는 못했지만
평문의 뒷부분이 정상적으로 보이는 현상이 있었습니다.

```text
앞쪽 블록
→ 깨진 문자열

뒤쪽 블록
→ 일부 필드와 숫자가 정상적으로 출력
```

CBC 모드에서는 잘못된 IV를 사용하면
첫 번째 평문 블록에 직접적인 영향을 주지만,
Key와 이후 암호문 블록이 정상이라면
뒤쪽 블록이 정상적으로 복호화될 수 있습니다.

따라서 다음과 같은 추론이 가능했습니다.

```text
뒤쪽 블록이 정상
    ↓
AES Key 후보는 맞을 가능성이 높음
    ↓
전체 알고리즘도 맞을 가능성이 높음
    ↓
첫 번째 블록에 영향을 주는 IV가 다를 가능성이 높음
```

이 결과를 통해 문제 범위를
Key 전체가 아니라 IV 구성 방식으로 더 좁힐 수 있었습니다.
</section>

<section class="case-study-section" aria-labelledby="finding-title" markdown="1">

## 8. 실제 적용 규격 확인 {#finding-title}

테스트 결과 실제 승인 데이터는
공식 가이드의 설명과 다음 부분에서 달랐습니다.

| 비교 항목 | 공식 가이드 | 실제 데이터에서 확인한 방식 |
|---|---|---|
| 암호문 입력 | HEX 문자열 | Base64 |
| RSA 결과 처리 | 문자열 변환 후 HEX 디코딩 | Raw Byte 그대로 사용 |
| AES Key 길이 | 16바이트 | RSA 결과 전체 32바이트 |
| AES 알고리즘 | AES-128-CBC | AES-256-CBC |
| IV | Key와 동일 | 문서에 없던 별도의 고정 IV |
| 결과 | 패딩 오류 | 정상 승인정보 복호화 |

실제 고정 IV 값과 키 정보는
포트폴리오에 공개하지 않습니다.

중요한 것은 보안값 자체를 알아낸 것이 아니라,
공식 문서와 실제 연동 규격 사이에 다음 불일치가 있음을
재현 가능한 테스트로 확인한 점입니다.

```text
인코딩 방식 불일치
RSA 결과 해석 방식 불일치
AES Key 길이 불일치
IV 규격 누락
```
</section>

<section class="case-study-section" aria-labelledby="verification-title" markdown="1">

## 9. 하나의 데이터로 결론 내리지 않고 교차 검증 {#verification-title}

특정 승인 데이터 한 건에서만 성공하는 방식은
실제 규격이라고 단정하기 어렵습니다.

따라서 서로 다른 카드사와 여러 승인 데이터에
동일한 복호화 방식을 적용했습니다.

```text
카드사 A 승인 데이터
→ 정상 복호화

카드사 B 승인 데이터
→ 정상 복호화

카드사 C 승인 데이터
→ 정상 복호화
```

다음 조건이 모두 충족되는지 확인했습니다.

- 같은 암호문 디코딩 방식 사용
- 같은 RSA 결과 처리 방식 사용
- 같은 AES Key 길이 사용
- 같은 IV 정책 사용
- 카드사와 관계없이 평문 구조 일치
- 실제 승인 흐름 정상 완료

이를 통해 특정 테스트 데이터에만 맞춘
우연한 결과가 아니라는 점을 검증했습니다.
</section>

<section class="case-study-section" aria-labelledby="implementation-title" markdown="1">

## 10. 확인한 규격을 애플리케이션 경계에 반영 {#implementation-title}

확인한 복호화 규격이
서비스 코드 여러 곳에 퍼지지 않도록
복호화 책임을 별도의 컴포넌트로 분리했습니다.

```java
public interface ApprovalPayloadDecryptor {

    DecryptedApprovalPayload decrypt(
        EncryptedApprovalPayload payload
        );
}
```

```java
@Component
public class PaycoApprovalPayloadDecryptor
implements ApprovalPayloadDecryptor {

    private final PrivateKeyProvider privateKeyProvider;
    private final ApprovalPayloadParser payloadParser;

    @Override
    public DecryptedApprovalPayload decrypt(
        EncryptedApprovalPayload payload
        ) {
        byte[] encryptedAesKey =
            Base64.getDecoder().decode(
            payload.encryptedKey()
            );

        byte[] aesKey =
            decryptAesKey(
            encryptedAesKey,
            privateKeyProvider.get()
            );

        byte[] cipherText =
            Base64.getDecoder().decode(
            payload.encryptedData()
            );

        String plainText = decryptPayload(
            cipherText,
            aesKey,
            fixedIv()
            );

        return payloadParser.parse(plainText);
    }
}
```

실제 코드에서는 키와 IV를
소스 코드에 직접 노출하지 않도록 관리합니다.

```text
개발환경
→ 개발환경 전용 키

운영환경
→ 별도 생성한 운영환경 전용 키
```

복호화가 끝난 이후에는
외부 응답 문자열을 그대로 비즈니스 로직에 전달하지 않고
검증된 결과 객체로 변환합니다.

```java
public record DecryptedApprovalPayload(
    String approvalNumber,
    String cardNumber,
    String acquirerCode,
    boolean firstPayment
    ) {
}
```
</section>

<section class="case-study-section" aria-labelledby="additional-title" markdown="1">

## 11. 복호화 외 누락된 응답 규격도 함께 확인 {#additional-title}

복호화에 성공한 이후
실제 승인 응답에는 공식 문서에 없는 필드가 포함되어 있음을 확인했습니다.

예를 들어 최초 결제 여부와 관련된 필드가
응답 데이터에 존재했지만 가이드에는 설명이 없었습니다.

또한 일부 카드사의 매입사 코드 정보도
문서에서 보완이 필요한 상태였습니다.

따라서 외부 기관에 다음 내용을 함께 요청했습니다.

- 복호화 규격의 공식 확인
- AES Key 길이와 IV 규격 반영
- 실제 암호문 인코딩 방식 반영
- 누락된 승인 응답 필드 설명 추가
- 일부 카드사 매입사 코드 가이드 보완

문제 하나만 임시로 해결하고 종료하지 않고,
같은 문서를 사용하는 다음 개발자가
동일한 문제를 겪지 않도록 규격 보완까지 요청했습니다.
</section>

<section class="case-study-section" aria-labelledby="communication-title" markdown="1">

## 12. 외부 기관에 재현 가능한 분석 결과 전달 {#communication-title}

외부 기관에 단순히 “복호화가 안 된다”고 전달하면
애플리케이션 구현 문제인지 공식 규격 문제인지 판단하기 어렵습니다.

따라서 다음 구조로 분석 결과를 정리했습니다.

```text
1. 재현되는 현상
2. 공식 가이드 기준 구현 결과
3. 검토한 가설
4. 일부 성공한 중간 결과
5. 최종 성공 조건
6. 카드사별 실데이터 검증 결과
7. 공식 가이드와 실제 규격 비교표
8. 누락된 응답 필드
9. 가이드 수정 요청
```

특히 공식 가이드와 실제 적용 방식을
항목별 표로 비교했습니다.

```text
입력 인코딩
RSA 결과 처리
AES 알고리즘
AES Key
IV
복호화 결과
```

이렇게 정리함으로써
외부 기관도 차이를 빠르게 재현하고
내부 담당자에게 전달할 수 있었습니다.
</section>

<section class="case-study-section" aria-labelledby="result-title" markdown="1">

## 13. 결과 {#result-title}

분석 결과를 전달한 이후
외부 기관에서도 해당 방식으로 복호화가 되는 것을 확인했습니다.

이후 다음 내용이 반영된 수정 가이드가 전달되었습니다.

- 실제 복호화 규격 보완
- 누락된 응답 파라미터 추가
- 일부 카드사 매입사 코드 정보 보완
- 개발환경 키 사용 방식 확인
- 운영환경 별도 키 사용 협의

최종적으로 다음 결과를 만들었습니다.

### 연동 일정 지연 방지

공식 가이드 수정만 기다리지 않고
직접 규격을 검증하여 후속 개발과 테스트를 진행할 수 있었습니다.

### 복호화 로직 정상화

여러 카드사의 실제 승인 데이터에서
같은 규칙으로 정상 복호화되는 것을 확인했습니다.

### 공식 문서 개선

개별 프로젝트의 임시 우회 코드로 끝내지 않고,
공식 가이드의 잘못된 규격과 누락된 필드가 수정되도록 했습니다.

### 운영 키 분리

개발환경과 운영환경에서
서로 다른 키를 사용하는 방향으로 보안 경계를 구분했습니다.
</section>

<section class="case-study-section" aria-labelledby="difficulty-title" markdown="1">

## 14. 이 문제에서 어려웠던 부분 {#difficulty-title}

이 사례에서 어려웠던 점은
AES 복호화 코드를 작성하는 것 자체가 아니었습니다.

Java의 암호화 API 사용 방법은
일반적인 문서와 예제를 통해 확인할 수 있습니다.

실제 난점은 다음과 같았습니다.

### 신뢰해야 할 문서가 틀릴 수 있다는 판단

공식 문서가 존재하는 상황에서는
자신의 구현이 잘못됐다고 판단하기 쉽습니다.

하지만 동일한 실패가 반복되고
입력 데이터의 특성과 예제 구조가 맞지 않는다는 점을 근거로
문서와 실제 규격의 불일치 가능성을 검토했습니다.

### 정답을 모르는 상태에서 변수 통제

입력 형식, Key, IV를 동시에 변경하지 않고
각 변수를 분리하여 결과를 비교했습니다.

### 부분적으로 정상인 결과 해석

첫 번째 블록만 깨지고 이후 데이터가 정상인 결과를
단순 실패로 버리지 않고,
CBC 모드의 특성을 기준으로 IV 문제 가능성을 추론했습니다.

### 기술 결과를 외부 기관이 검증할 수 있는 형태로 설명

개인적인 추측이나 코드 전체를 전달하는 대신
가이드와 실제 규격의 차이를 비교표와 검증 결과로 전달했습니다.
</section>

<section class="case-study-section" aria-labelledby="tradeoff-title" markdown="1">

## 15. 트레이드오프와 주의사항 {#tradeoff-title}

### 확인한 규격을 공식 표준으로 단정할 수 없음

외부 기관의 공식 확인 전까지
분석한 방식은 실데이터를 기반으로 한 검증 결과였습니다.

따라서 코드에 반영하더라도
외부 기관에 규격 확인과 문서 수정을 함께 요청해야 했습니다.

### 알려진 평문 구조에 대한 의존

평문의 필드 구조를 이용하면
복호화 후보의 정확도를 높일 수 있습니다.

다만 특정 문자열 하나만으로 성공 여부를 판단하지 않고,
필드 구조와 여러 데이터에 대한 교차 검증이 필요합니다.

### 보안정보 공개 제한

포트폴리오에서는 다음 정보를 공개하지 않습니다.

- 실제 개인키
- 공개키 원문
- 실제 암호문
- 실제 고정 IV 값
- 운영환경 키 관리 방식의 세부사항
- 내부 거래 식별자
- 실제 승인 데이터

이 사례의 목적은 보안값을 공개하는 것이 아니라,
불완전한 연동 규격을 어떻게 검증했는지 설명하는 것입니다.

### 역분석 결과를 장기 계약으로 사용하지 않음

직접 확인한 규격은 연동을 진행하기 위한 근거가 되었지만,
장기적으로는 수정된 공식 가이드를
시스템 간 계약의 기준으로 사용해야 합니다.
</section>

<section class="case-study-section" aria-labelledby="retrospective-title" markdown="1">

## 16. 회고 {#retrospective-title}

이 문제는 처음에는 단순한 복호화 오류로 보였습니다.

하지만 공식 가이드의 예제를 반복 수정하는 방식으로는
원인을 찾을 수 없었습니다.

그래서 문제를 다음과 같이 다시 정의했습니다.

```text
“왜 패딩 오류가 발생하는가?”
```

가 아니라,

```text
“공식 문서의 각 암호화 조건 중
실제 데이터와 일치하지 않는 것은 무엇인가?”
```

로 바꾸었습니다.

그 이후에는 복호화 과정을 독립된 변수로 나누고,
같은 데이터에 여러 후보를 반복 적용할 수 있는 테스트를 만들었습니다.

```text
입력 인코딩
    ↓
RSA 결과 처리
    ↓
AES Key 길이
    ↓
IV
    ↓
평문 구조 검증
```

부분적으로 복호화된 결과도 버리지 않고
CBC 블록의 특성을 이용해 다음 가설을 세웠고,
여러 카드사의 실데이터로 결과를 반복 검증했습니다.

최종적으로는 개발 일정 안에 연동을 완료했을 뿐 아니라,
분석 내용을 외부 기관에 전달하여
공식 문서의 잘못된 복호화 규격과 누락 필드가 수정되도록 했습니다.

이 경험을 통해 외부 연동에서 중요한 것은
문서를 그대로 구현하는 것만이 아니라,

- 문서와 실제 데이터가 일치하는지 검증하고
- 실패 조건을 재현 가능한 테스트로 만들며
- 가능성을 독립된 가설로 분리하고
- 결과를 외부 조직이 확인할 수 있는 형태로 설명하는 것

이라고 배웠습니다.
</section>

</article>
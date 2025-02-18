## 3줄 요약

- `Mockito`의 `stubbing` 과정에서는 모든 인자에 대해 일관된 매처 사용이 필수이며, 실제 객체와 `Matchers`를 혼용하면 내부 기록 시 예외가 발생합니다.
- `when()`은 메서드 호출을 가로채 인자 정보를 내부에 저장하여 이후 `thenThrow()` 등으로 `stubbing` 동작을 설정하는 준비 단계입니다.
- `verify()`는 별도의 검증 모드에서 호출 횟수와 순서를 확인하며, 미완료된 `verification` 상태가 있으면 예외를 발생시켜 사용자 실수를 알려줍니다.


## 문제 상황

아래와 같이 하나의 인자만 `Matchers`를 사용하지 않은 상태로 설정하였을 때, 예외가 발생했습니다.

![image.png](https://github.com/user-attachments/assets/50567fa3-baa1-4f16-b4d3-14caf45f48ab)

항상 `Mock`을 사용할 때 이런 예외 상황들을 자주 만났지만, `any()` 키워드 대신 실제 객체를 넣어주는 것으로 항상 예외 상황을 우회했었습니다.🙃

이번 기회에 이러한 문제 상황이 왜 생기는 것이고, 정상적인 경우에는 어떻게 구성되는지 알아보고자 합니다.

![image.png](https://github.com/user-attachments/assets/3d79e339-09aa-4338-8e7c-b5b542d97313)

![image.png](https://github.com/user-attachments/assets/fc1cfe14-1b41-4390-a9d9-438ae278c7b1)

![image.png](https://github.com/user-attachments/assets/9efaca6d-8a8f-42c0-83cd-8aad1a2db705)

결론적으로 `Mockito`는 메서드 호출을 가로채고, 인자 정보(여기서는 `reservation`과 `any(PaymentResult.class)` 매처)를 기록하여 `stubbing`을 위한 내부 상태에 저장합니다.
- 아래 사진과 같이 `confirmReservationPayment` 메서드 호출이 진행되지 않더라도 인자들을 캡쳐하여 내부에 기록하는 과정에서 예외가 발생합니다.

    ![image.png](https://github.com/user-attachments/assets/f9df0aa5-6703-4b00-b6ce-55950b02ef3d)

>Mockito는 stubbing을 위해 전달된 인자들을 캡처하여 내부에 기록하는데, 이때 모든 인자에 대해 일관되게 Matchers를 사용해야 합니다. 하나의 인자는 실제 객체(예, reservation)로, 다른 하나는 Matchers(예, any(PaymentResult.class))로 전달하면 내부적으로 매처의 개수가 일치하지 않아 예외가 발생합니다.


## 그렇다면 정상적인 경우에는 어떤 호출이 이루어질까?

1. **메서드 호출 인터셉트**
    - `when(...)` 구문 안에 있는 `reservationService.confirmReservationPayment(reservation, any(PaymentResult.class))` 호출은 실제로 실행되지 않습니다.
    - 대신, Mockito는 해당 호출을 가로채고, 인자 정보(여기서는 `reservation`과 `any(PaymentResult.class)` 매처)를 기록하여 `stubbing`을 위한 내부 상태에 저장합니다.
        - `앞선 문제는 이 과정에서 예외가 발생했습니다.`
2. **Stubbing 등록**
    - 이후에 `thenThrow(...)`를 호출함으로써, 나중에 `reservationService.confirmReservationPayment(...)`가 호출될 때 `Mockito`가 지정한 예외를 던지도록 `stubbing`(행동 설정)을 등록합니다.

따라서, **실제로 위의 코드를 실행할 때 내부의 `confirmReservationPayment` 메서드가 실행되지 않고, 대신 그 호출이 캡처되어 이후 호출 시 `Mockito`가 해당 예외를 던지도록 설정됩니다.**

단, 만약 `reservationService`가 **spy** 객체인 경우에는, 기본적으로 실제 메서드가 호출될 수 있으므로 주의해야 합니다. -> [@SpyBean 언제 써야 하나요?](https://ho-tea.github.io/blog/?post=%5B20240205%5D_%5B%5D_%5BTest%5D_%5Bspy.jpeg%5D_%5B%EC%9D%B4%EB%A6%84+%EA%B7%B8%EB%8C%80%EB%A1%9C+%EC%8A%A4%ED%8C%8C%EC%9D%B4%EA%B0%80+%EC%96%B8%EC%A0%9C+%ED%95%84%EC%9A%94%ED%95%9C%EC%A7%80%EC%97%90+%EB%8C%80%ED%95%B4+%EC%95%8C%EC%95%84%EB%B4%85%EB%8B%88%EB%8B%A4.%5D_%5B%5D.md)

- 순수한 **mock** 객체라면 위와 같이 메서드가 실행되지 않고 stubbing만 기록됩니다.
- 스파이(부분 모의 객체)의 경우, 실제 메서드 호출이 일어나지 않도록 하려면 `doThrow(...).when(spy).confirmReservationPayment(...)`와 같은 구문을 사용해야 합니다.


![image.png](https://github.com/user-attachments/assets/d682f420-0ecd-43f4-8fe5-c55ab3a8f243)

- `mockingProgress.stubbingStarted();`는 stubbing 프로세스가 시작되었음을 알리고,
    - **stubbing 모드의 시작을 알림** 
        - 이후에 호출되는 메서드 호출(예: `when(…)` 안의 메서드 호출)을 캡처해서 stubbing 정보를 저장할 수 있도록 내부 상태를 설정합니다.
    
    따라서, 이 메서드는 stubbing 정보를 **저장하기 위한 준비** 단계로 볼 수 있습니다.
    
    즉, `stubbing`을 기록(저장)할 수 있도록 준비하는 역할을 하며, `stubbing` 자체를 실행하는 것은 아닙니다.
    
- 그 뒤에 실제 모의 객체의 메서드 호출이 이루어지면, 그 호출 정보가 내부적으로 캡처되어 `pullOngoingStubbing()`을 통해 가져와서 stubbing을 설정하는 데 사용됩니다.
    - **기록된 stubbing 정보 추출**
        - `pullOngoingStubbing()`은 위에서 캡처한 stubbing 정보를 내부 저장소(ThreadLocal 같은 곳)에 저장된 상태에서 "꺼내는(pull)" 역할을 합니다.
        - 반환되는 값은 `OngoingStubbing<T>` 객체로, 이 객체에 대해 `thenReturn()`, `thenThrow()` 등의 추가 stubbing 설정을 이어서 적용할 수 있습니다.

이렇게 함으로써, `when(...)` 메서드는 사용자가 stubbing하려는 메서드 호출을 올바르게 캡처하고, 이후에 `thenReturn`, `thenThrow` 등과 같이 모의 행동을 정의할 수 있게 됩니다.

![image.png](https://github.com/user-attachments/assets/d8733209-1daa-4fa9-b578-c68d67f5b657)

- **글로벌 설정 검증**
    - `GlobalConfiguration.validate()`를 호출하여 Mockito의 전역 설정이 올바른지 확인합니다.

![image.png](https://github.com/user-attachments/assets/84124a53-bcfd-4456-87e4-a8f6876dccd4)


- **미완료된 Verification 체크**
    - 만약 `this.verificationMode`가 `null`이 아니라면, 즉 사용자가 verification을 시작했으나 정상적으로 마무리하지 않은 상태라면,
        - `this.verificationMode.getLocation()`을 통해 verification이 시작된 위치 정보를 가져오고,
        - `this.verificationMode`를 `null`로 초기화한 후,
        - `Reporter.unfinishedVerificationException(location)`을 통해 미완료된 verification에 대한 예외를 던집니다.
    - **해당 과정은 `verify()`를 사용하는 경우에 의미 있습니다.**
        >Verification (verify() 사용) -> 
        verify()는 모의 객체의 메서드가 예상대로 호출되었는지 검증하는 단계입니다.
        이때 verificationMode(예: times(1), never() 등)를 사용하여 호출 횟수나 순서를 지정할 수 있습니다.
        만약 verification을 시작했지만 제대로 마무리하지 않으면, 내부적으로 미완료된 verification에 대해 예외를 던져 사용자가 검증을 완전히 수행하지 않았음을 알려줍니다.
- **Argument Matchers 상태 검증**
    - 만약 verification 모드가 설정되어 있지 않다면,
        - `this.getArgumentMatchersStorage().validateState()`를 호출하여, stubbing이나 verification 과정에서 사용된 `argument Matchers`들이 올바른 상태(ex: 모두 사용되었거나, 미완료 상태가 없는지)를 확인합니다.


`when`을 호출한 상태는 `verification` 모드가 설정되어 있지 않으므로, `this.getArgumentMatchersStorage().validateState()`를 호출합니다.


![image.png](https://github.com/user-attachments/assets/154ed5c0-63c6-46a1-bdad-1db0d1721ba0)

![image.png](https://github.com/user-attachments/assets/50ebecae-c2c5-475c-a3e7-e78eb1830711)


![image.png](https://github.com/user-attachments/assets/9bf955a6-f44b-45d9-9290-14d3ce3e9c89)

간단하게 디버깅 도구를 활용해서 어떠한 순서로 메서드들이 호출/처리 되는지 알아보았습니다.

가끔은 이렇게 테스트에 통과하여도 `Mockito`와 같은 라이브러리를 사용한다면, 직접 디버깅을 찍어보면서 어떠한 순서로 메서드들이 호출되고 처리되는지를 알아보는 과정도 재밌는 것 같습니다. 🤔
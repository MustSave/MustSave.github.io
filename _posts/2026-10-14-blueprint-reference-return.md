---
layout: post
title: "분명 참조를 반환했는데, Blueprint는 복사하고 있었다"
date: 2026-10-14 00:00:00 +0900
categories: [Unreal Engine]
tags: [cpp, blueprint, uht]
permalink: /posts/blueprint-reference-return/
mermaid: true
---

Unreal Engine에서 C++로 함수를 작성하다 보면 배열이나 구조체를 참조로 반환하는 경우가 있습니다. 기존 데이터를 복사하지 않고 사용하거나, 호출한 쪽에서 원본을 수정하려는 목적입니다. 그런데 같은 함수를 `UFUNCTION(BlueprintCallable)`로 노출하면 C++의 반환 형식만 보고 Blueprint에서도 같은 객체를 참조할 것이라고 판단하기 쉽습니다.

이 차이를 확인하려고 배열 멤버를 `[1, 2, 3, 4, 5]`로 초기화하고, Blueprint에서도 직접 가져올 수 있도록 프로퍼티로 노출했습니다.

```cpp
UPROPERTY(BlueprintReadOnly)
TArray<uint8> ArrayProperty;
```

같은 멤버를 일반 참조와 const 참조로 반환하는 함수도 준비했습니다. 두 함수 모두 반환 형식에 `&`가 있고, C++에서는 `ArrayProperty`를 참조합니다.

```cpp
UFUNCTION(BlueprintCallable)
TArray<uint8>& GetArrayRef() { return ArrayProperty; }

UFUNCTION(BlueprintCallable)
const TArray<uint8>& GetConstArrayRef() const { return ArrayProperty; }
```

## 참조를 반환했는데 원본이 바뀌지 않았다

Blueprint가 받은 배열이 원본과 같은 객체인지 비교하려고, 배열의 주소와 원소 목록을 출력하는 함수를 만들었습니다. 여기서 `&Array`는 원소 데이터의 시작 주소가 아니라 `TArray` 객체 자체의 주소입니다.

```cpp
UFUNCTION(BlueprintCallable)
static void PrintAddressAndItem(const TArray<uint8>& Array)
{
    UE_LOG(LogTemp, Log, TEXT("Array address: %p"), &Array);
    UE_LOG(LogTemp, Log, TEXT("Array items: [%s]"),
        *FString::JoinBy(Array, TEXT(", "), [](uint8 Item)
        {
            return FString::FromInt(Item);
        }));
}
```

비교 기준이 되는 원본 주소는 객체 내부에서 멤버 배열을 출력해 확인했습니다. 반환값이나 프로퍼티 노드를 거치지 않고 `ArrayProperty`를 출력 함수에 직접 넘깁니다.

```cpp
UFUNCTION(BlueprintCallable)
void PrintMyArrayPropertyAddressAndItem() const
{
    PrintAddressAndItem(ArrayProperty);
}
```

Editor Utility Blueprint에서 테스트 객체를 만들고, `Sequence`로 네 경로를 차례대로 실행했습니다. 먼저 `GetArrayRef()`의 반환값을 출력한 뒤 `Add` 노드로 `99`를 추가했습니다. 이어서 `GetConstArrayRef()`의 반환값, `ArrayProperty`를 직접 가져온 값, 마지막으로 객체 내부에서 멤버를 출력한 값을 비교했습니다.

![참조 반환 함수, const 참조 반환 함수, 프로퍼티 직접 접근과 객체 내부 출력을 비교하는 Blueprint 테스트](/assets/posts/blueprint-reference-return/reference_return_test.png)

C++에서 `GetArrayRef()`를 직접 호출해 반환된 참조에 원소를 추가하면 멤버 배열이 바뀝니다. Blueprint에서도 같은 방식으로 원본을 바꾼다면, 첫 번째 출력 이후에는 `99`가 추가된 배열을 볼 것으로 예상했습니다. 하지만 이 테스트에서는 이후의 출력에서도 멤버 배열의 값이 그대로였고, 함수의 반환값을 출력한 주소도 멤버 주소와 달랐습니다.

로그에 나타난 배열 주소와 값은 다음과 같았습니다.

```text
0000020CBB9165A0  [1, 2, 3, 4, 5]
0000020CBB916590  [1, 2, 3, 4, 5]
0000020CBB9148B0  [1, 2, 3, 4, 5]
0000020CBB9148B0  [1, 2, 3, 4, 5]
```

각 출력이 어느 경로에 대응하는지 놓고 보면 다음과 같습니다.

| 실행 순서 | 출력한 대상 | 배열 객체 주소 | 원소 |
| --- | --- | --- | --- |
| Then 0 | `GetArrayRef()` 반환값 | `0000020CBB9165A0` | `[1, 2, 3, 4, 5]` |
| Then 1 | `GetConstArrayRef()` 반환값 | `0000020CBB916590` | `[1, 2, 3, 4, 5]` |
| Then 2 | `ArrayProperty` 직접 접근 | `0000020CBB9148B0` | `[1, 2, 3, 4, 5]` |
| Then 3 | 객체 내부에서 멤버 출력 | `0000020CBB9148B0` | `[1, 2, 3, 4, 5]` |

첫 번째 경로에서 원소를 추가했지만 뒤의 경로에서 출력한 원본은 바뀌지 않았습니다. 프로퍼티를 직접 가져온 경우에는 객체 내부에서 출력한 멤버와 같은 주소가 나왔습니다. 함수의 반환값을 연결했을 때와 프로퍼티를 직접 가져왔을 때의 차이를 이해하려고, Unreal Header Tool이 생성한 함수 호출 코드를 확인했습니다.

## 생성 코드에 있는 복사 대입

Unreal Header Tool은 C++ 선언을 읽고 Unreal의 리플렉션 시스템에 필요한 코드를 생성합니다. Blueprint에서 C++ 함수를 호출하는 경로에도 이런 생성 코드가 사용됩니다. 예제의 `GetArrayRef()`에 대응하는 코드에는 다음 대입이 있었습니다. [Unreal Header Tool 문서](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-header-tool-for-unreal-engine)

```cpp
*(TArray<uint8>*)Z_Param__Result = P_THIS->GetArrayRef();
```

오른쪽의 `GetArrayRef()`는 멤버 배열의 참조를 반환합니다. 하지만 왼쪽은 그 참조를 그대로 보관하는 변수가 아니라, 호출 결과를 담을 공간에 놓인 `TArray<uint8>`입니다. 반환된 배열을 이 결과 객체에 대입하면서 복사가 일어납니다.

이 동작을 풀어 쓰면 다음과 같습니다.

```cpp
TArray<uint8>& Result =
    *static_cast<TArray<uint8>*>(Z_Param__Result);

Result = P_THIS->GetArrayRef();
```

여기서 `Result`가 참조라는 것은 결과 저장 공간에 있는 배열을 가리킨다는 뜻입니다. `Result = ...`는 참조가 가리키는 대상을 바꾸는 연산이 아니라 그 배열 객체에 값을 대입하는 연산입니다. 오른쪽 표현식은 lvalue이므로 이 경로에서는 `TArray`의 복사 대입을 사용합니다. `TArray`는 값 의미를 가지므로 복사된 배열의 원소를 바꾸어도 원래 배열은 바뀌지 않습니다. [TArray의 복사와 대입](https://dev.epicgames.com/documentation/en-us/unreal-engine/array-containers-in-unreal-engine)

```mermaid
flowchart LR
  A[UObject의 멤버 배열] -->|C++ 함수가 참조 반환| B[생성된 호출 코드]
  B -->|복사 대입| C[결과 저장 공간의 배열]
  C --> D[Blueprint가 사용하는 반환값]
```

함수 선언에서는 참조를 반환하지만, Blueprint 호출 경로에는 결과 객체에 값을 옮기는 단계가 추가되는 것입니다. 예제에서 원본이 바뀌지 않았던 현상은 이 생성 코드와 연결해서 설명할 수 있었습니다.

## 이벤트로 배열을 전달한 경우

배열을 이벤트로 전달하는 경우도 비교했습니다. 같은 객체의 Multicast Delegate에 두 Custom Event를 연결하고, 멤버 배열을 전달한 뒤 각 이벤트에서 주소를 출력했습니다. 이벤트 호출이 끝난 다음에는 객체 내부의 멤버 주소를 출력했습니다.

![멤버 배열을 Multicast Delegate의 두 Custom Event에 전달하고 객체 내부의 배열 주소와 비교하는 Blueprint 테스트](/assets/posts/blueprint-reference-return/delegate_test.png)

```text
LogTemp: Array address: 0000023177F65C30
LogTemp: Array items: [1, 2, 3, 4, 5]
LogBlueprintUserMessages: [None] 1
LogTemp: Array address: 0000023177F65C40
LogTemp: Array items: [1, 2, 3, 4, 5]
LogBlueprintUserMessages: [None] 2
LogTemp: Array address: 000002314C4B18A8
LogTemp: Array items: [1, 2, 3, 4, 5]
```

이 테스트에서도 두 이벤트가 받은 배열의 주소와 객체 내부에서 출력한 멤버 배열의 주소가 달랐습니다. 배열을 넘겼다는 사실만으로 호출받은 쪽이 원본 배열 객체를 그대로 다룬다고 판단하기 어려웠습니다.

## 프로퍼티 직접 접근과 함수 반환을 구분하기

이 테스트에서 Blueprint에 노출된 배열 프로퍼티를 직접 가져와 출력 함수에 전달했을 때는, 객체 내부에서 멤버를 출력한 경우와 같은 주소가 나왔습니다. 이 경로에서는 별도의 반환값 배열을 만드는 대신 원본 배열 객체를 참조하고 있었습니다. 반면 C++ 함수가 배열 참조를 반환하는 경로에서는 생성된 호출 코드가 결과 저장 공간에 배열을 복사했습니다.

따라서 Blueprint에서 배열을 읽을 때도 프로퍼티에 직접 접근하는 것과 참조 반환 함수를 호출하는 것을 구분해야 합니다. 함수 선언에 `&`가 있다고 프로퍼티 직접 접근과 같은 전달 방식이 되는 것은 아닙니다. 또한 `BlueprintReadOnly`는 Blueprint에서 프로퍼티를 쓸 수 있는지를 제한하는 지정자이므로, 이번에 관찰한 참조 전달을 그 지정자 자체가 보장한다고 해석해서는 안 됩니다.

원본 배열을 바꾸려는 목적이라면 참조 반환값을 외부에서 수정하는 방식에 의존하기보다, 객체 안에서 멤버 배열을 수정하는 함수를 제공하는 구성을 검토할 수 있습니다. C++ 안에서 처리할 수 있는 작업은 원본 참조로 직접 다루고, Blueprint에는 필요한 연산을 노출하면 호출 결과를 복사한 배열과 원본 배열을 혼동하기도 어렵습니다.

반환값이 작거나 호출이 드문 경우에는 복사가 문제가 되지 않을 수도 있습니다. 반대로 큰 배열을 자주 반환하는 경로라면 선언의 `&`만 보고 복사를 피했다고 판단하기보다, 생성된 호출 코드에서 결과가 어떻게 전달되는지 확인할 필요가 있습니다.

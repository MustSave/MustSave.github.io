---
layout: post
title: "싸피레이스에서 학습은 PyTorch로, 추론은 NumPy로 나눈 이유"
date: 2026-10-13 00:00:00 +0900
permalink: /posts/ssafy-race/
mermaid: true
---

싸피레이스 주행 알고리즘 대회에 참여하면서 PPO로 차량을 주행시키는 방법을 시도했습니다. 대회에서는 NumPy를 사용할 수 있었는데, 이 규칙을 보고 학부 때 NumPy로 CNN을 직접 구현했던 경험이 떠올랐습니다. 그래서 처음에는 학습부터 추론까지 NumPy로 구현해 제출 코드에서도 그대로 사용하는 방향으로 작업을 시작했습니다.

## 학습까지 NumPy로 구현할 필요가 있을까

구현을 진행하다 보니 신경망의 출력을 계산하는 것뿐 아니라 역전파와 옵티마이저까지 직접 만들어야 했고, 작업이 점점 복잡해졌습니다. 이 과정에서 제출 환경에 실제로 필요한 기능을 다시 살펴보았습니다. 차량이 주행하는 동안에는 모델을 새로 학습할 필요 없이, 이미 학습한 모델에 차량 상태를 넣고 행동을 결정하면 됐습니다.

그렇다면 학습 코드까지 제출 환경에 맞출 필요는 없었습니다. 학습은 PyTorch로 진행하고, 제출할 코드에는 NumPy로 구현한 추론 네트워크만 넣기로 했습니다. 대신 PyTorch로 학습한 가중치를 NumPy 네트워크로 옮길 수 있어야 했으므로, 학습과 추론 사이에 가중치 변환 과정을 추가했습니다.

전체 흐름은 다음과 같습니다.

```mermaid
flowchart LR
    A["시뮬레이터에서 PyTorch PPO 학습"] --> B["학습 가중치 추출"]
    B --> C["NumPy 네트워크로 가중치 변환"]
    C --> D["저장·Base64 인코딩"]
    D --> E["제출 코드에 포함"]
    E --> F["차량 상태로부터 행동 결정"]
```

## 학습한 가중치를 NumPy로 옮기기

학습이 끝난 모델에서 `state_dict`를 꺼내 NumPy 네트워크에 옮긴 뒤, 변환한 가중치를 파일로 저장했습니다. 이 과정을 담당한 `torch_to_numpy.py`의 핵심 부분은 다음과 같습니다.

```python
state_dict = ppo_torch.policy.state_dict()

ppo_numpy.load_from_torch_statedict(state_dict)

ppo_numpy.save(numpy_save_path)
```

가중치를 옮길 때는 두 구현에서 배열을 사용하는 방식도 맞춰야 했습니다. PyTorch의 Linear는 weight를 `(out_features, in_features)` 형태로 저장하고, 계산할 때 이를 전치해 입력과 곱합니다. 제가 작성한 NumPy Linear는 입력에 weight를 바로 곱하는 구조였으므로, PyTorch에서 가져온 weight를 전치해서 저장했습니다. [PyTorch Linear 문서](https://docs.pytorch.org/docs/2.9/generated/torch.nn.Linear.html)

입력 3개를 받아 출력 2개를 만드는 작은 계층을 예로 들면, 가중치의 형태는 다음과 같이 바뀝니다.

```text
PyTorch에서 가져온 weight       NumPy에 저장할 weight
[[1, 2, 3],                    [[1, 4],
 [4, 5, 6]]                     [2, 5],
                                [3, 6]]

형태: (2, 3)                   형태: (3, 2)
```

이렇게 전치하면 입력 3개에 가중치를 곱해 출력 2개를 계산할 수 있습니다.

`ppo_numpy.py`에서 actor의 가중치를 옮기는 코드는 다음과 같습니다. critic도 같은 방식으로 처리했습니다.

```python
for k in state_dict.keys():
    network, index, key = k.split(".")
    v = state_dict[k]
    if network == "actor":
        self.actor.modules[int(index)]._parameters[key].value = v.numpy().T if key == "weight" else v.numpy()
```

코드에서 weight에는 `.T`로 전치를 적용하고, bias는 그대로 전달했습니다. 이렇게 각 계층의 값을 옮겨 NumPy 네트워크에서 사용할 가중치를 구성했습니다.

당시 제출 조건은 단일 파일, 최대 200KB였습니다. 추론 코드와 가중치를 한 파일에 담기 위해 저장한 가중치를 Base64로 인코딩해 제출 코드에 포함했습니다. 주행할 때는 이 가중치를 불러온 NumPy 네트워크에 차량 상태를 입력하고, 선택된 행동에 따라 조향과 가속 등을 제어하도록 구성했습니다.

## 주행 결과

이렇게 구성한 모델로 기본 트랙 주행에 성공했습니다. 당시 주행 모습을 볼 수 있도록 녹화 영상에서 10초씩 발췌했습니다. 첫 번째는 직선에서 곡선 구간으로 이어지는 장면이고, 두 번째는 곡선 구간을 주행하는 장면입니다.

![싸피레이스 주행 장면 1](/assets/posts/ssafy-race/race_drive_01.gif)

첫 번째 녹화의 10~20초 구간. [전체 영상](/assets/posts/ssafy-race/race_drive_01.mp4)

![싸피레이스 주행 장면 2](/assets/posts/ssafy-race/race_drive_02.gif)

두 번째 녹화의 15~25초 구간. [전체 영상](/assets/posts/ssafy-race/race_drive_02.mp4)

## 시작 위치를 바꿔본 시도

장애물 트랙을 학습할 때는 시작 위치도 바꿔보았습니다. 트랙 초반이 곧은 직선 구간이었는데, 매번 이 구간부터 주행하는 점이 학습에 영향을 주는 것은 아닐까 생각했습니다. 그래서 학습을 시작하는 위치를 무작위로 바꾸는 방식을 적용해봤습니다.

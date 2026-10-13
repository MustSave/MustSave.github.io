---
layout: post
title: "Perforce 파일 수정 전에 체크아웃을 연결한 Hook"
date: 2026-10-13 00:00:00 +0900
permalink: /posts/perforce-hook/
mermaid: true
---

Perforce를 사용하는 개발 환경에서 Codex로 코드를 수정하다 보면, 파일이 읽기 전용으로 되어 있어 작업이 멈추곤 했습니다. 그때마다 직접 `p4 edit`을 실행하거나 프롬프트에 체크아웃을 요청해야 했습니다. 파일을 수정할 때 필요한 절차였지만, 같은 요청을 반복하면서 작업 흐름이 끊기는 것이 불편했습니다.

이 절차를 파일 수정 도구 앞에 연결해보기로 했습니다. 도구가 실행되기 전에 대상 파일을 알아낼 수 있다면, 그 파일을 먼저 체크아웃한 뒤 수정을 진행하도록 구성할 수 있다고 생각했습니다.

## 수정 직전에 파일 열기

Perforce의 `p4 edit`은 작업공간의 파일을 편집 상태로 열고 쓰기 권한을 활성화합니다. 따라서 수정 도구가 파일을 바꾸기 전에 이 명령을 실행하도록 했습니다. [Perforce p4 edit 문서](https://help.perforce.com/helix-core/server-apps/cmdref/current/Content/CmdRef/p4_edit.html)

도구 호출 전에 실행되는 `PreToolUse`에 Python 스크립트를 연결했습니다. 스크립트는 도구의 입력에서 파일 경로를 수집하고, 패치가 들어오면 수정·삭제 대상 경로를 읽었습니다. 상대 경로는 현재 작업 디렉터리를 기준으로 해석하고 중복을 제거한 뒤, 실제로 존재하는 파일에 `p4 edit`을 실행했습니다. [OpenAI Hooks 문서](https://learn.chatgpt.com/docs/hooks?translationFallback=de-DE)

예를 들어 수정 대상이 `Source/Example.cpp`라면, Hook이 이 경로를 읽어 다음 명령을 먼저 실행합니다.

```shell
p4 edit Source/Example.cpp
```

체크아웃이 끝나면 수정 도구가 같은 파일을 편집합니다. 사용자가 파일마다 체크아웃을 요청하던 절차를 이 순서로 연결했습니다.

처리 순서를 정리하면 다음과 같습니다.

```mermaid
flowchart LR
    A["파일 수정 도구 호출"] --> B["PreToolUse"]
    B --> C["대상 경로 수집·중복 제거"]
    C --> D["기존 파일에 p4 edit 실행"]
    D --> E["파일 수정 도구 실행"]
```

## 업무에 적용하기

체크아웃에 실패하면 명령 출력과 실패한 파일 경로를 남기도록 했습니다. 이 Hook을 당시 업무에 사용하고, 동작 방식도 팀원들에게 공유했습니다.

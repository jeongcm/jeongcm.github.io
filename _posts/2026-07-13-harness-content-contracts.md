---
title: "이직 에이전트를 하네스로 설계하기 (4) — 글의 모양도 운영 계약이다"
date: 2026-07-13 11:47:00 +0900
categories: [building-the-agent]
tags: [ai-agent, harness, blog, workflow, slack]
---

앞선 글에서는 하네스의 핵심이 자동화 자체보다 운영 기준을 계속 선명하게 만드는 일이라고 썼다. 그 뒤 실제로 블로그 초안을 만들고, Slack으로 보내고, 발행 직전 화면을 확인하면서 기준 하나가 더 추가됐다. 글의 내용뿐 아니라 **글이 어떤 카테고리에 들어가고, 어떤 형태로 렌더링되는지도 운영 계약의 일부**라는 점이다.

처음에는 기술 글을 하나의 큰 묶음으로 다뤘다. 데이터 엔지니어링 글도, Spring WebFlux 같은 백엔드 글도 모두 기술 지식이라는 점에서는 비슷해 보였다. 하지만 블로그를 포트폴리오처럼 쓰려면 글의 분류가 곧 나의 포지셔닝이 된다. Kafka, Flink, Iceberg, Airflow는 `data-engineering`으로 묶고, Spring WebFlux, Reactor, Kotlin backend, API 설계 같은 글은 `technical-knowledge > backend-knowledge` 아래에 두기로 했다. 백엔드 기반에서 데이터 플랫폼으로 확장해온 방향을 블로그 구조에서도 보이게 만든 것이다.

카테고리 정리는 단순한 태그 정리가 아니었다. 자동화 프롬프트, `AGENT.md`, 블로그 플레이북, 검증 기준, 기존 발행 글의 front matter까지 함께 바꿔야 했다. 한 파일만 고치면 다음 실행에서 다시 예전 기준이 튀어나온다. 하네스에서 기준은 코드 한 줄보다 넓게 퍼져 있다. 그래서 운영 문서, 롤 카드, 검증 체크리스트, 자동화 프롬프트가 같은 말을 하도록 맞추는 일이 중요했다.

이번에 더 선명해진 기준은 시각 자료였다. 글이 텍스트만 있으면 읽는 힘이 떨어져서 Mermaid 다이어그램을 넣기 시작했다. 그런데 실제 블로그에서는 Mermaid가 그림이 아니라 코드처럼 풀려 보였다. 초안 단계에서는 그럴듯했지만, 발행 화면에서는 독자가 읽기 불편한 노이즈가 됐다. 그래서 기본값을 바꿨다. 앞으로는 직접 작성한 SVG/PNG, inline SVG, 또는 직접 작성한 최소 코드 예시를 우선하고, Mermaid는 렌더링 확인이 된 경우에만 사용한다.

<figure class="post-diagram">
  <svg viewBox="0 0 980 320" role="img" aria-labelledby="contract-flow-title contract-flow-desc" xmlns="http://www.w3.org/2000/svg">
    <title id="contract-flow-title">Harness content contract flow</title>
    <desc id="contract-flow-desc">A request updates the operating contract, draft, verifier rules, Slack review, and public post in a controlled loop.</desc>
    <defs>
      <marker id="contract-arrow" markerWidth="12" markerHeight="12" refX="10" refY="6" orient="auto">
        <path d="M2,2 L10,6 L2,10 Z" fill="#46637f" />
      </marker>
      <style>
        .contract-box { fill: #f8f5ea; stroke: #46637f; stroke-width: 2; rx: 14; }
        .contract-accent { fill: #e8eef6; stroke: #46637f; stroke-width: 2; rx: 14; }
        .contract-label { font: 700 17px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #263238; }
        .contract-small { font: 13px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; fill: #5b6470; }
        .contract-line { stroke: #46637f; stroke-width: 3; fill: none; marker-end: url(#contract-arrow); }
        .contract-loop { stroke: #8a6f3d; stroke-width: 3; fill: none; stroke-dasharray: 8 8; marker-end: url(#contract-arrow); }
      </style>
    </defs>

    <rect x="30" y="92" width="135" height="82" class="contract-box" />
    <text x="98" y="126" text-anchor="middle" class="contract-label">Request</text>
    <text x="98" y="150" text-anchor="middle" class="contract-small">new rule</text>

    <rect x="210" y="92" width="150" height="82" class="contract-accent" />
    <text x="285" y="126" text-anchor="middle" class="contract-label">AGENT.md</text>
    <text x="285" y="150" text-anchor="middle" class="contract-small">operating contract</text>

    <rect x="405" y="92" width="150" height="82" class="contract-box" />
    <text x="480" y="126" text-anchor="middle" class="contract-label">Playbook</text>
    <text x="480" y="150" text-anchor="middle" class="contract-small">writing rules</text>

    <rect x="600" y="92" width="150" height="82" class="contract-box" />
    <text x="675" y="126" text-anchor="middle" class="contract-label">Draft</text>
    <text x="675" y="150" text-anchor="middle" class="contract-small">front matter + SVG</text>

    <rect x="795" y="92" width="150" height="82" class="contract-accent" />
    <text x="870" y="126" text-anchor="middle" class="contract-label">Slack</text>
    <text x="870" y="150" text-anchor="middle" class="contract-small">review / publish</text>

    <path d="M165 133 H210" class="contract-line" />
    <path d="M360 133 H405" class="contract-line" />
    <path d="M555 133 H600" class="contract-line" />
    <path d="M750 133 H795" class="contract-line" />

    <path d="M870 174 C870 270 285 270 285 174" class="contract-loop" />
    <text x="575" y="292" text-anchor="middle" class="contract-small">published output feeds back into the next operating rule</text>
  </svg>
  <figcaption>하네스에서는 화면에서 발견한 문제도 다음 실행의 운영 규칙으로 되돌아간다.</figcaption>
</figure>

이 작은 문제는 좋은 피드백이었다. 에이전트는 마크다운 파일만 보면 "다이어그램을 넣었다"고 판단할 수 있다. 하지만 사용자가 보는 것은 렌더링된 페이지다. 그래서 검증 기준도 바뀌어야 했다. `completion-criteria`에는 "보조 자료가 들어갔는가"뿐 아니라 "코드처럼 노출되지 않는가"가 들어갔다. verifier 롤 카드도 Mermaid가 그대로 보이면 실패로 보도록 바꿨다. 산출물의 품질은 파일 내용만이 아니라 실제 표시 형태까지 포함한다.

Slack의 역할도 다시 확인됐다. Slack은 단순 알림 채널이 아니라 발행 직전의 검토 표면이다. 초안은 Slack으로 보내고, 그 안에서 어떤 글을 발행할지 결정한다. 다만 Slack 메시지를 보냈다고 해서 곧바로 공개되는 것은 아니다. 실제 발행은 `/career blog publish <draft-file>` 같은 명시적인 명령 뒤에만 진행된다. 이 구분 덕분에 글을 고치고 다시 Slack으로 보내는 흐름이 자연스럽게 가능했다.

이번 수정에서 흥미로웠던 점은, "글 하나 고치기"가 금방 운영 기준 수정으로 번졌다는 것이다. WebFlux 글의 다이어그램이 코드처럼 보였고, Kafka 글도 같은 방식으로 작성되어 있었다. 그래서 두 글의 Mermaid를 inline SVG로 바꾸고, `drafts`, `published`, `blog/site/_posts`를 모두 맞췄다. 이어서 앞으로 같은 문제가 반복되지 않도록 `AGENT.md`, 플레이북, 롤 카드, 검증 기준, 자동화 프롬프트까지 갱신했다.

이 과정을 지나며 하네스의 역할이 더 분명해졌다. 하네스는 나 대신 블로그를 "마구" 발행하는 도구가 아니다. 내가 한 번 판단한 기준을 다음 실행에서도 유지하게 만드는 장치다. 어떤 글은 객관적인 기술 해설이어야 하고, 어떤 글은 하네스 운영 회고여야 한다. 어떤 글은 `data-engineering`으로, 어떤 글은 `technical-knowledge > backend-knowledge`로 들어가야 한다. 어떤 시각 자료는 안전하지만, 어떤 표현 방식은 실제 블로그에서 깨질 수 있다.

결국 개인용 에이전트 하네스의 품질은 똑똑한 답변보다 **반복 가능한 판단**에서 나온다. 한 번의 대화에서 고친 내용을 다음 실행의 규칙으로 남길 수 있어야 한다. 문서가 오래되면 자동화도 오래된 판단을 반복한다. 반대로 문서, 플레이북, 검증 기준, 자동화 프롬프트가 함께 갱신되면 작은 시행착오가 하네스의 운영 지식으로 쌓인다.

이번 4편의 결론은 단순하다. 글의 카테고리, 시각 자료의 렌더링, Slack 승인 경로처럼 사소해 보이는 것들도 모두 운영 계약이다. 자동화가 안정적으로 보이려면 이런 작은 계약들이 계속 최신 상태여야 한다. 하네스 엔지니어링은 결국 기능을 더 많이 붙이는 일이 아니라, 사용 중 발견한 어긋남을 다음 실행의 기준으로 바꾸는 일에 가깝다.

---
title: "이직 에이전트를 하네스로 설계하기 (4) — Claude Code로 블로그 디자인을 고쳐보기"
date: 2026-07-13 11:47:00 +0900
categories: [building-the-agent]
tags: [ai-agent, harness, claude-code, blog-design, workflow]
---

앞선 글에서는 하네스의 핵심이 자동화 자체보다 운영 기준을 계속 선명하게 만드는 일이라고 썼다. 이번에는 조금 더 눈에 보이는 작업을 했다. Claude Code를 이용해 GitHub Pages 블로그의 홈 화면과 글 표현 방식을 직접 손봤다. 단순히 글을 자동으로 쓰는 데서 끝내지 않고, 그 글이 놓이는 공간의 디자인까지 에이전트와 함께 다듬어 본 것이다.

처음 블로그는 글 목록이 있는 평범한 기술 블로그에 가까웠다. Claude Code로 `blog/site/_layouts/home.html`과 커스텀 SCSS를 함께 보면서, 첫 화면에 매일 바뀌는 개발자 명언, GitHub Activity, 카테고리 묶음, 자주 보는 기술 문서 링크, 최신 업데이트 글이 보이도록 구성했다. 긴 hero를 두기보다 실제 개발 블로그처럼 읽을거리와 활동 신호가 먼저 보이게 하는 쪽을 선택했다.

이 과정에서 재미있었던 점은, 디자인 수정이 단순 CSS 작업으로 끝나지 않았다는 것이다. 홈 화면에서 어떤 카테고리가 보이는지, 글 카드에서 카테고리 칩이 어떻게 읽히는지, 코드블록과 inline code가 어색하지 않은지까지 보게 됐다. 블로그를 포트폴리오처럼 쓰려면 글의 내용만큼이나 분류와 배치가 중요했다. 그래서 Kafka, Flink, Iceberg, Airflow는 `data-engineering`으로 묶고, Spring WebFlux, Reactor, Kotlin backend, API 설계 같은 글은 `technical-knowledge > backend-knowledge` 아래에 두기로 했다.

카테고리 정리는 단순한 태그 정리가 아니었다. Claude Code가 한 파일만 고치면 다음 실행에서 다시 예전 기준이 튀어나올 수 있다. 그래서 자동화 프롬프트, `AGENT.md`, 블로그 플레이북, 검증 기준, 기존 발행 글의 front matter까지 함께 바꿨다. 하네스에서 기준은 코드 한 줄보다 넓게 퍼져 있다. 운영 문서, 롤 카드, 검증 체크리스트, 자동화 프롬프트가 같은 말을 하도록 맞추는 일이 디자인 작업의 일부가 됐다.

이번에 더 선명해진 기준은 시각 자료였다. 글이 텍스트만 있으면 읽는 힘이 떨어져서 Mermaid 다이어그램을 넣기 시작했다. 그런데 실제 블로그에서는 Mermaid가 그림이 아니라 코드처럼 풀려 보였다. 초안 단계에서는 그럴듯했지만, 발행 화면에서는 독자가 읽기 불편한 노이즈가 됐다. Claude Code로 글 파일과 공개 `_posts`를 같이 확인하면서, 기본값을 직접 작성한 SVG/PNG, inline SVG, 또는 직접 작성한 최소 코드 예시로 바꿨다. Mermaid는 렌더링 확인이 된 경우에만 사용하기로 했다.

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

이 작은 문제는 좋은 피드백이었다. 에이전트는 마크다운 파일만 보면 "다이어그램을 넣었다"고 판단할 수 있다. 하지만 사용자가 보는 것은 렌더링된 페이지다. Claude Code를 디자인 수정 도구로 쓰려면 diff만 보는 것으로는 부족하다. 글 카드, 카테고리, 코드블록, 다이어그램이 실제 화면에서 어떻게 보일지까지 기준에 넣어야 한다. 그래서 `completion-criteria`에는 "보조 자료가 들어갔는가"뿐 아니라 "코드처럼 노출되지 않는가"가 들어갔다.

Slack의 역할도 다시 확인됐다. Slack은 단순 알림 채널이 아니라 발행 직전의 검토 표면이다. 초안은 Slack으로 보내고, 그 안에서 어떤 글을 발행할지 결정한다. 다만 Slack 메시지를 보냈다고 해서 곧바로 공개되는 것은 아니다. 실제 발행은 `/career blog publish <draft-file>` 같은 명시적인 명령 뒤에만 진행된다. 이 구분 덕분에 글을 고치고 다시 Slack으로 보내는 흐름이 자연스럽게 가능했다.

사실 이 하네스의 출발점에는 채용 맞춤 이력서 기능도 있었다. 회사별 JD를 읽고, 정본 이력서와 상세 경력 근거를 조합해 맞춤 이력서와 매칭 리포트를 만드는 구조다. 다만 지금까지의 실제 사용을 돌아보면 이 기능은 매일 쓰는 자동화라기보다, 지원하고 싶은 회사가 생겼을 때 꺼내 쓰는 온디맨드 기능에 가깝다. 반대로 블로그는 매일 초안, Slack 검토, 발행, 화면 확인이 반복되면서 기준이 빠르게 다듬어졌다. 하네스 안에 기능이 있다고 해서 모두 같은 빈도로 성숙하는 것은 아니었다.

이 점도 운영 기준으로 남길 필요가 있다. 채용 맞춤 이력서는 여전히 중요하지만, 아무 공고나 자동으로 긁어와서 지원서를 만드는 방향은 아니다. 내가 지원하고 싶은 회사를 고르면, 그때 JD를 기준으로 `master-resume`, ABC 상세 근거, 제출용 경력기술서를 재조합한다. 자동화의 역할은 지원을 대신하는 것이 아니라, 선택한 회사에 맞게 내 경험을 더 정확히 배열해 주는 것이다.

이번 수정에서 흥미로웠던 점은, "블로그 디자인을 고치기"가 금방 운영 기준 수정으로 번졌다는 것이다. 홈 화면을 정리하고, 카테고리 표시를 다듬고, WebFlux 글과 Kafka 글의 Mermaid를 inline SVG로 바꿨다. 그리고 `drafts`, `published`, `blog/site/_posts`를 모두 맞췄다. 이어서 앞으로 같은 문제가 반복되지 않도록 `AGENT.md`, 플레이북, 롤 카드, 검증 기준, 자동화 프롬프트까지 갱신했다.

이 과정을 지나며 하네스의 역할이 더 분명해졌다. 하네스는 나 대신 블로그를 "마구" 발행하는 도구가 아니다. 내가 한 번 판단한 기준을 다음 실행에서도 유지하게 만드는 장치다. 어떤 글은 객관적인 기술 해설이어야 하고, 어떤 글은 하네스 운영 회고여야 한다. 어떤 글은 `data-engineering`으로, 어떤 글은 `technical-knowledge > backend-knowledge`로 들어가야 한다. 어떤 시각 자료는 안전하지만, 어떤 표현 방식은 실제 블로그에서 깨질 수 있다.

결국 개인용 에이전트 하네스의 품질은 똑똑한 답변보다 **반복 가능한 판단**에서 나온다. 한 번의 대화에서 고친 내용을 다음 실행의 규칙으로 남길 수 있어야 한다. 문서가 오래되면 자동화도 오래된 판단을 반복한다. 반대로 문서, 플레이북, 검증 기준, 자동화 프롬프트가 함께 갱신되면 작은 시행착오가 하네스의 운영 지식으로 쌓인다.

이번 4편의 결론은 단순하다. Claude Code는 블로그 글을 쓰는 도구이기도 하지만, 블로그라는 제품의 작은 디자인 시스템을 함께 고치는 도구이기도 하다. 글의 카테고리, 홈 화면의 구성, 시각 자료의 렌더링, Slack 승인 경로처럼 사소해 보이는 것들도 모두 운영 계약이다. 하네스 엔지니어링은 결국 기능을 더 많이 붙이는 일이 아니라, 사용 중 발견한 어긋남을 다음 실행의 기준과 화면의 형태로 바꾸는 일에 가깝다.

---
title: "이직 에이전트 하네스로 설계하기 (5) — 정답은 hermes agent?"
date: 2026-10-28 12:30:00 +09:00
categories: [building-the-agent]
tags: [ai-agent, harness, hermes, job-search, workflow]
---

이 연재 1~4편은 '이직 에이전트 하네스'를 제약에서 출발시켜 로그인, 예약, 승인, 운영 기준의 순서로 쌓아 왔다. 결론은 하나로 모인다. 개인용 에이전트의 안정성은 더 똑똑한 프롬프트가 아니라, 반복 가능한 판단을 다음 실행에도 유지하는 데서 나온다. 4편까지 겪은 어긋남을 그대로 둔 채 기능만 추가하면 하네스는 낡은 판단을 반복하는 도구가 된다.

이전 하네스에서는 기준이 세 곳에 흩어져 있었다. AGENT.md의 운영 기준, 자동화 프롬프트, 검증 체크리스트다. 세션이 바뀌면 프롬프트의 일부만 살아남고, 나머지는 매번 '어디를 참조하지?'로 돌아갔다. 더 큰 문제는 검증이다. 에이전트가 만든 산출물을 에이전트가 스스로 '괜찮다'고 판단하는 구조는 약해서, 사실성·과장·공개 경계·승인 문제를 다음 실행으로 그대로 넘겼다.

핵심은 기준을 '기억'이 아니라 '파일'로 내리는 일이었다. 실행 환경이 달라져도 같은 판단을 유지하려면 기준을 프롬프트의 일부가 아니라 파일의 형태로 고정해야 했다.

이전 하네스에서는 기준을 파일로 분산하는 방법이 약했다. 디자인 수정 작업에서 확인했듯 에이전트는 마크다운 파일만 보고 '다이어그램을 넣었다'고 판단할 수 있지만 사용자가 보는 것은 렌더링된 페이지다. 코드블록, 카테고리 칩, SVG와 Mermaid가 화면에서 어떻게 보이는지를 알지 못하면 기준에 적어둔 규칙은 언젠가 어긋난다. 이력서 쪽에서도 같은 문제가 반복됐다. 정본 이력서, 상세 근거, 제출용 경력기술서가 한 폴더에 흩어져 있어도 작업자가 어느 파일을 참조해야 하는지 매 실행마다 다시 정해야 했다.

개인용 에이전트를 '어디서 어떻게 쓸지'까지 포함해 한 덩어리로 묶는 허브로 보면 답은 분명해진다. 기준은 파일로 고정하고 역할은 분리하고 승인은 사람이 건넌다. hermes agent의 스킬 체계는 동작을 사양으로 고정하고, 작업자 스킬은 초안을 만들고, 검증자 스킬은 completion-criteria로 확인하는 역할 분리를 그대로 구현한다. 스킬은 필요할 때만 불러들이는 점진적 공개 구조라서 운영 기준도 파일 전체로 끌어오는 대신 필요할 때 필요한 만큼만 데려올 수 있다.

예약과 승인 흐름도 허브의 라우팅으로 정리된다. 스케줄과 승인은 게이트웨이의 메시징 플랫폼과 cron Tick에서 담당하고, 모델이 쉬운 작업과 어려운 작업을 자동으로 나누는 라우팅은 단순 작업에 소모되던 맥락을 남긴다. 이로써 남는 것은 판단을 구동하는 맥락이 아니라 판단을 은퇴시키는 기준과 그 기준을 다시 읽는 파일이다.

남은 과제는 세 가지다. 스킬 저장소와 블로그 `_posts` 사이의 카테고리 일치 기준 강화, 승인 흐름의 자동화 강화, 이력서 부문 자동화 기준 안정화. 이런 구조는 1~4편에서 산발적으로 쌓였던 어긋남을 한곳으로 모은다. 기준을 파일로 고정하면 다음 실행이 같은 판단을 읽고, 어긋남이 생겼을 때 어디를 고치면 되는지 명확해진다. 하네스의 품질은 이제 모델의 성능이 아니라, 기준이 얼마나 일관되게 파일에 남는가로 잰다.

---

## 하네스 설계의 변화

이전 하네스는 기준을 프롬프트로 들고 다녔고, 허브 모델에서는 기준을 파일로 고정한다.

<svg xmlns="http://www.w3.org/2000/svg" width="980" height="280" viewBox="0 0 980 280" role="img" aria-labelledby="hub-title hub-desc">
  <title id="hub-title">이직 에이전트 하네스 설계: 기준 고정화</title>
  <desc id="hub-desc">이전 하네스는 기준을 프롬프트로 들고 다녔고, 허브 모델에서는 기준을 파일로 고정한다.</desc>
  <style>
    rect { rx: 10; }
    .user { fill: #f8f5ea; stroke: #46637f; stroke-width: 2; }
    .hub  { fill: #eef7ff; stroke: #3b82f6; stroke-width: 2; }
    .wkr  { fill: #e6f7e6; stroke: #10b981; stroke-width: 2; }
    .ver  { fill: #fef3c7; stroke: #f59e0b; stroke-width: 2; }
    .appr { fill: #f3f4f6; stroke: #6b7280; stroke-width: 2; }
    .pub  { fill: #eef2ff; stroke: #6366f1; stroke-width: 2; }
    .t { fill: #263238; font: 700 15px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; }
    .t2 { fill: #5b6470; font: 12px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; }
    .t3 { fill: #1e40af; font: 700 14px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; }
    .t4 { fill: #1e40af; font: 12px -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; }
    .arrow { stroke: #9ca3af; stroke-width: 2; fill: none; }
    .loop { stroke: #8a6f3d; stroke-width: 3; fill: none; stroke-dasharray: 8 8; }
  </style>
  <rect class="user" x="20" y="55" width="150" height="120"/>
  <text class="t" x="95" y="90">사용자</text>
  <text class="t2" x="95" y="112">지시 + 승인</text>
  <rect class="hub" x="210" y="70" width="190" height="120"/>
  <text class="t3" x="305" y="110">허브 (hermes agent)</text>
  <text class="t4" x="305" y="134">스킬 저장소 + 라우팅 + 세션</text>
  <rect class="wkr" x="450" y="55" width="150" height="110"/>
  <text class="t" x="525" y="95">작업자 스킬</text>
  <text class="t2" x="525" y="117">초안 생성</text>
  <rect class="ver" x="450" y="135" width="150" height="110"/>
  <text class="t" x="525" y="175">검증자 스킬</text>
  <text class="t2" x="525" y="197" font-size="11">completion-criteria 확인</text>
  <rect class="appr" x="660" y="85" width="160" height="90"/>
  <text class="t" x="740" y="125">승인</text>
  <text class="t2" x="740" y="147">사람 게이트</text>
  <rect class="pub" x="850" y="85" width="100" height="90"/>
  <text class="t" x="900" y="125">게시</text>
  <text class="t2" x="900" y="147">_posts</text>
  <path class="arrow" d="M170 115 H210"/>
  <path class="arrow" d="M400 105 H450"/>
  <path class="arrow" d="M400 135 H450"/>
  <path class="arrow" d="M600 110 H660"/>
  <path class="arrow" d="M600 180 H660"/>
  <path class="arrow" d="M820 130 H850"/>
  <path class="loop" d="M900 175 V250 H160"/>
  <text x="530" y="270" text-anchor="middle" class="t2">다음 실행 운영 기준으로 피드백</text>
</svg>

---

[개념 예시]

```text
┌─ 작업자 스킬 ─┐   ┌─ 검증자 스킬 ─┐   ┌─ 승인 게이트 ─┐
│ 초안 생성       │ → │ completion-criteria 확인 │ → │ 승인 전 대기    │
│ (블로그/이력서) │   │ (사실성·과장·공개 경계) │   │ (사람 판단)    │
└────────────────┘   └────────────────┘   └───────────────┘
```

---

[참고]
- Hermes Agent GitHub: https://github.com/NousResearch/hermes-agent
- Hermes Agent 스킬 문서: https://github.com/NousResearch/hermes-agent/tree/main/website/docs/user-guide/features/skills.md
- Hermes Agent 아키텍처: https://github.com/NousResearch/hermes-agent/blob/main/website/docs/developer-guide/architecture.md

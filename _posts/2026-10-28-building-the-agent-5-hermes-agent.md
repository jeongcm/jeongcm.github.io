---
title: "이직 에이전트를 하네스로 설계하기 (5) — 블로그 자동화 Hermes Agent 로"
date: 2026-10-28 12:30:00 +09:00
categories: [building-the-agent]
tags: [ai-agent, harness, hermes, ec2, slack, github-pages, workflow]
---

매일 아침 기술 블로그 초안을 만들고 Slack으로 받아보는 작업은 이미 하고 있었다. 공식 자료를 찾고, 한국어로 정리하고, 검토한 글을 GitHub Pages에 올리는 흐름이다. 여기에 Hermes Agent를 도입한 이유는 이 작업을 계속 실행할 장소와, 초안에 대한 수정·승인을 이어갈 창구가 필요했기 때문이다.

이번에는 EC2에서 Hermes를 실행하고 Slack 봇에 '야꾸'라는 이름을 붙였다. 먼저 옮길 대상으로 고른 것은 매일 쓰던 기술 블로그였다. 모델을 연결하는 데서 시작했지만, 실제로 정해야 했던 것은 누가 글을 만들고, 어디에 저장하며, 어떤 승인을 받아 공개할 것인지였다.

## 이미 자동화는 있었지만, 운영하는 곳은 나뉘어 있었다

[이전 글](/posts/harness-content-contracts/)까지 만든 하네스에는 작성 플레이북, 검증 기준, 초안 폴더, 공개 블로그 저장소가 있었다. Slack 알림과 승인 후 발행을 담당하는 Go 봇도 따로 운영했다. 작성 규칙을 파일로 관리하는 방식은 그때부터 사용하고 있었다.

반복 작업은 꽤 구체적이었다. 최근 글을 보고 데이터 엔지니어링과 백엔드 지식 중 부족한 분야를 고르고, 공식 자료를 확인한 뒤 초안을 작성했다. 일반 기술 글은 개인 경험을 지어내지 않는 객관적인 해설체로, 이 연재는 실제 운영 과정을 돌아보는 글로 구분했다. 시각 자료도 직접 작성한 SVG나 코드 예시를 사용하도록 정해 두었다.

다만 초안을 만드는 대화와 예약 작업, 알림을 보내는 도구, 글을 발행하는 봇은 각각 연결해야 했다. 10월 27일에는 초안 파일을 완성한 뒤에도 Slack 전송이 승인 검토에서 막혔다. 파일 경로와 요약을 외부로 보내는 권한이 확인되지 않았다는 이유였고, 다시 전송을 승인한 뒤에야 알림이 나갔다. 글 한 편을 만드는 일과 그 결과를 전달하는 일의 상태가 서로 달랐던 셈이다.

내가 원한 사용 흐름은 분명했다. 오전에 초안을 받아보고, 같은 대화에서 고칠 부분을 말하고, 내용이 확정되면 발행을 승인하는 것이다. 노트북을 계속 켜 두는 대신 상시 실행할 EC2를 두고, 내가 들어가는 창구를 Slack으로 모으기로 했다.

## 처음 옮길 일은 매일 쓰던 블로그였다

Hermes와 함께 사용할 수 있는 Oh My Hermes도 살펴봤다. 여러 작업을 나누거나 모델과 스킬을 연결하는 기능은 흥미로웠지만, 이번 도입의 출발점은 이미 반복하고 있던 블로그 작업이었다. 확장 패키지를 어디까지 쓸지보다 Hermes의 모델 연결, 파일 작업, Slack 대화부터 확인할 필요가 있었다.

EC2에서 설치를 진행할 때도 각 선택의 의미를 하나씩 확인했다. Nous Portal이 안내한 인증 링크는 Mac 브라우저에서 열고, EC2 터미널은 인증을 기다리도록 두는 방식이었다. 원격 서버에서 로그인한다고 해서 그 서버에 브라우저 화면까지 띄울 필요는 없었다. 현재 공식 문서는 Nous Portal을 device-code 인증 대상으로 안내한다. [원격 인증 문서](https://hermes-agent.nousresearch.com/docs/guides/oauth-over-ssh)

도구 선택은 맡길 작업에서 출발했다. 기술 자료를 조사하려면 웹 검색과 본문 추출이 필요하다. 이미지 생성과 음성 입력은 별도 용도이고, 웹페이지를 클릭하거나 입력해야 할 때는 브라우저 자동화가 필요하다. 메뉴에 있는 도구를 전부 켜는 것보다 첫 작업에 필요한 기능을 구분하는 과정이 먼저였다.

터미널 백엔드의 `Local`도 확인했다. 여기서 로컬은 Hermes가 실행되는 EC2였다. 이 설정에서 파일을 쓰고 명령을 실행하면 EC2 안에서 동작한다. 내 Mac의 블로그 파일이나 기존 예약 설정이 저절로 옮겨지는 것은 아니었다.

## Slack 봇에 야꾸라는 이름을 붙였다

Slack 연결은 Hermes가 생성한 앱 매니페스트를 바탕으로 준비했다. 앱 이름은 '야꾸', 매니페스트의 봇 표시 이름은 `yakku`로 정했다. 생성 과정에서 표시 이름의 문자 제한도 확인했다. `/model`, `/stop` 같은 슬래시 명령어는 Hermes가 처리하는 이름을 유지했다. 봇 이름을 바꾸는 것과 명령어 체계를 바꾸는 것은 별개의 설정이었다.

연결에는 Bot Token과 App-Level Token이 필요했다. Hermes의 Slack 연결은 Socket Mode를 사용하므로 EC2에 Slack 요청을 받는 공개 HTTP 주소를 추가할 필요가 없다. Slack을 접점으로 삼는 구성에서 편리한 부분이었다. [Slack 연결 문서](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/slack)

이후 야꾸가 공개키를 생성했다는 응답과 SSH 설정 변경 승인 요청을 Slack에서 받았다. 이때부터 대화는 설치법을 묻는 단계를 넘어, 서버에서 어떤 작업을 실행하려는지 보고 결정하는 단계로 이어졌다. 설정 파일을 수정할 권한과 실제 글을 발행할 권한도 구분해야 했다. GitHub 연결을 확인한다고 해서 글을 push하는 것까지 승인한 것은 아니었다.

## 초안을 쓰는 권한과 발행하는 권한을 나누었다

블로그 작업을 옮기면서 가장 먼저 생긴 질문은 “블로그 저장소도 EC2에 있어야 하나?”였다. 초안만 작성한다면 Markdown을 저장할 폴더로 충분하다. 기존 글과 주제가 겹치는지 확인하고, 승인된 글을 실제 사이트에 반영하려면 공개 블로그 저장소를 EC2에서도 사용할 수 있어야 했다.

그래서 초안과 운영 기록을 보관할 공간, 공개 사이트에 반영할 저장소를 분리하는 구조를 잡았다. 기존 작업 폴더에는 이력서와 비공개 자료도 있어 전체를 그대로 옮기는 방식은 피하기로 했다. 블로그 운영에 필요한 작성 규칙과 공개 글 이력부터 가져오는 것이 범위에 맞았다.

<figure>
<svg xmlns="http://www.w3.org/2000/svg" width="760" height="390" viewBox="0 0 760 390" role="img" aria-labelledby="yakku-flow-title yakku-flow-desc" style="display:block;max-width:100%;height:auto;background:#f8fafc;border-radius:12px">
<title id="yakku-flow-title">EC2의 야꾸와 Slack을 연결한 블로그 운영 구조</title>
<desc id="yakku-flow-desc">EC2에서 초안을 작성하고 Slack에서 검토와 명시 승인을 받은 뒤 승인된 버전을 GitHub에 반영한다. GitHub Actions와 Pages의 배포 결과를 Slack으로 전달하는 목표 흐름이다.</desc>
<defs><marker id="yakku-flow-arrow" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto"><path d="M0,0 L0,6 L9,3 z" fill="#475569"/></marker></defs>
<g font-family="sans-serif" text-anchor="middle" fill="#0f172a">
<text x="380" y="35" font-size="19" font-weight="700">작성·검토·발행의 책임을 연결하기</text>
<rect x="30" y="65" width="200" height="85" rx="10" fill="#dbeafe"/>
<text x="130" y="99" font-size="18" font-weight="700">EC2 · 야꾸</text><text x="130" y="127" font-size="15">자료 조사와 초안 저장</text>
<rect x="280" y="65" width="200" height="85" rx="10" fill="#e2e8f0"/>
<text x="380" y="99" font-size="18" font-weight="700">Slack · 검토</text><text x="380" y="127" font-size="15">본문 확인과 수정 요청</text>
<rect x="530" y="65" width="200" height="85" rx="10" fill="#fef3c7"/>
<text x="630" y="99" font-size="18" font-weight="700">사람 · 발행 승인</text><text x="630" y="127" font-size="15">초안 ID와 버전 지정</text>
<rect x="530" y="225" width="200" height="85" rx="10" fill="#dbeafe"/>
<text x="630" y="259" font-size="18" font-weight="700">EC2 · 발행 작업</text><text x="630" y="287" font-size="15">승인 파일만 commit·push</text>
<rect x="280" y="225" width="200" height="85" rx="10" fill="#dcfce7"/>
<text x="380" y="259" font-size="18" font-weight="700">GitHub</text><text x="380" y="287" font-size="15">Actions 빌드 → Pages</text>
<rect x="30" y="225" width="200" height="85" rx="10" fill="#e2e8f0"/>
<text x="130" y="259" font-size="18" font-weight="700">Slack · 결과</text><text x="130" y="287" font-size="15">배포 상태와 공개 URL</text>
<text x="380" y="356" font-size="15">예약 작업의 완료와 공개 배포의 완료는 따로 확인한다.</text>
</g>
<g fill="none" stroke="#475569" stroke-width="2" marker-end="url(#yakku-flow-arrow)"><path d="M230 108 H270"/><path d="M480 108 H520"/><path d="M630 150 V215"/><path d="M530 268 H490"/><path d="M280 268 H240"/></g>
</svg>
<figcaption>직접 작성한 목표 운영 구조. 매일 예약 실행과 버전별 승인 절차의 반복 동작은 계속 확인해야 한다.</figcaption>
</figure>

발행 규칙에는 승인 대상을 명확히 하는 내용을 넣기로 했다. “최신 글 올려줘”만으로 처리하면 그사이 초안이 추가되거나 수정됐을 때 검토한 내용과 공개되는 내용이 달라질 수 있다. 초안 ID와 버전을 보여주고, 그 버전의 승인을 받은 뒤 반영하는 방식이 필요했다. 승인 이후 본문이 바뀌면 다시 검토받도록 하는 것도 같은 이유다.

기술적으로는 야꾸가 공개 저장소에 승인된 글을 넣고 commit·push하면 기존 GitHub Actions가 빌드와 Pages 배포를 담당한다. EC2에는 글을 만드는 실행 환경을 두고, 사이트 제공은 기존 GitHub Pages에 맡기는 구성이다. 원격 반영에 성공했는지, 빌드가 성공했는지, 공개 URL에 글이 보이는지는 각각 확인해야 한다.

다만 이 규칙을 문서에 적는 것만으로 서버 차원의 접근 통제가 생기지는 않는다. 쓰기 인증을 가진 에이전트에게 “승인 후 발행”을 지시하는 것과, 승인된 요청만 받아 실행하는 별도 발행 서비스를 두는 것은 보장 수준이 다르다. 이번 도입에서는 그 차이를 남겨 두고 운영 절차부터 정리했다.

## 첫 글은 올라갔고, 내용에서 다음 과제가 드러났다

도입 과정을 다룬 이 글의 첫 버전이 실제 블로그에 올라왔다. 저장소에는 글을 추가한 커밋과 SVG 표현을 수정한 후속 커밋이 남았다. 적어도 초안이 공개 게시물로 이어진 결과물은 생겼다.

그런데 내용을 읽고 다시 수정하기로 했다. 첫 버전은 파일 기반 기준이나 역할 분리를 추상적으로 설명하는 데 많은 분량을 썼다. 기존에도 파일로 운영 기준을 관리했다는 사실, 왜 실행 장소를 EC2로 옮기려 했는지, Slack에서 어떤 대화를 이어가고 싶었는지가 충분히 드러나지 않았다. 도구의 기능 설명은 있었지만 도입 경험을 읽을 수 있는 구체적인 장면이 부족했다.

이것도 이번 도입의 결과였다. 글이 저장되고 발행됐다는 사실은 전달 경로가 작동했다는 근거가 되지만, 글의 내용에 만족한다는 의미까지 포함하지는 않는다. 사람의 검토를 남겨둔 이유가 첫 게시물에서부터 드러났다. 이 수정본은 그 피드백을 반영해 필요성, 선택, 진행 과정과 확인된 결과를 다시 연결한 기록이다.

아직 성과로 단정하지 않을 부분도 있다. 매일 오전 9시 예약이 누락 없이 반복되는지, Slack의 수정 요청이 해당 초안 버전으로 이어지는지, 승인된 내용만 발행되는지, 서버 재시작 뒤에도 같은 흐름이 유지되는지는 반복 실행으로 확인해야 한다. 초안 하나가 발행된 결과만으로 이 조건들이 모두 검증됐다고 말할 수는 없다.

예약 결과를 받은 스레드에서 대화를 이어가려면 Hermes의 작업별 `attach_to_session` 설정도 확인해야 한다. 공식 문서상 기본 예약 알림과 후속 대화가 가능한 예약 결과는 구분된다. “Slack으로 보낸다”는 설정만으로 내가 기대한 검토 대화가 완성되는 것은 아니었다. [예약 작업 문서](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron#continuable-jobs-reply-to-a-cron-delivery)

이번에 만든 변화는 기존 작성 기준을 유지하면서 실행 장소와 대화 창구를 옮기는 것이었다. EC2의 야꾸에게 반복 준비 작업을 맡기고, 나는 Slack에서 내용을 읽고 수정과 발행을 결정하는 방향이다. 앞으로는 새 기능을 붙이기 전에 이 흐름이 다음 날에도, 수정이 여러 번 오간 뒤에도 같은 결과를 내는지 확인하려 한다.

## 참고 자료

- [Hermes Agent — Slack 연결](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/slack)
- [Hermes Agent — 예약 작업과 후속 대화](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron)
- [Hermes Agent — 원격 환경의 OAuth 인증](https://hermes-agent.nousresearch.com/docs/guides/oauth-over-ssh)
- [GitHub — Deploy key 관리](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys)

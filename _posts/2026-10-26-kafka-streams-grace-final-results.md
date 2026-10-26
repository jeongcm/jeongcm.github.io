---
title: "Kafka Streams grace period: 늦은 이벤트와 최종 결과의 경계"
date: 2026-10-26 07:00:00 +0900
categories: [data-engineering]
tags: [kafka-streams, event-time, window, grace-period, suppression]
---

Kafka Streams에서 5분 윈도우에 grace를 2분 지정해도 실제 시계가 7분 지났다는 이유만으로 결과가 확정되지는 않는다. Grace는 늦은 이벤트를 집계에 받아들이는 범위이고, 최종 결과만 내보내는 정책은 별도로 선택한다. 이 구분은 지연 데이터의 정확도와 알림의 응답 시간을 함께 설계할 때 중요하다.

## 시계가 아니라 데이터가 시간을 움직인다

공식 Core Concepts는 stream-time을 레코드 처리에 따라 전진하는 시간으로 설명한다. 입력 timestamp의 의미는 TimestampExtractor와 Kafka timestamp 설정에 달려 있다. 따라서 이벤트 발생 시각으로 집계하려면 어떤 필드가 시간의 기준인지 먼저 확정해야 한다.

TimeWindows.ofSizeAndGrace는 윈도우 크기와 종료 이후의 허용 지연을 명시한다. 예를 들어 [10:00, 10:05) 윈도우에 grace 2분을 적용하면 종료 기준은 10:07이다. 해당 집계의 stream-time이 10:06인 동안 도착한 timestamp 10:04 레코드는 반영할 수 있지만, 이미 10:08까지 전진했다면 이 윈도우에는 반영되지 않는다. ofSizeWithNoGrace는 이 허용 지연을 0으로 설정한다.

<figure>
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 760 230" role="img" aria-labelledby="grace-title grace-desc" style="width:100%;height:auto;background:#f8fafc;border-radius:12px">
<title id="grace-title">윈도우와 grace의 stream-time 기준</title>
<desc id="grace-desc">10시부터 10시 5분까지 집계 윈도우, 10시 7분까지 grace 구간이다. stream-time이 10시 6분일 때 timestamp 10시 4분 레코드는 반영되지만 10시 8분일 때는 거절된다.</desc>
<g font-family="sans-serif" fill="#0f172a">
<text x="30" y="32" font-size="18" font-weight="bold">Stream-time 기준 · 실제 시계 경과와 다름</text>
<rect x="40" y="60" width="350" height="50" rx="6" fill="#dbeafe"/>
<rect x="390" y="60" width="140" height="50" rx="6" fill="#fef3c7"/>
<text x="150" y="91" font-size="17">집계 윈도우</text>
<text x="435" y="91" font-size="17">grace</text>
<path d="M40 120 H710" stroke="#475569" stroke-width="2"/>
<text x="40" y="144" font-size="15">10:00</text>
<text x="370" y="144" font-size="15">10:05</text>
<text x="510" y="144" font-size="15">10:07</text>
<text x="640" y="144" font-size="15">10:08</text>
<text x="40" y="178" font-size="16">10:04 이벤트 도착 → stream-time 10:06: 반영 가능</text>
<text x="40" y="207" font-size="16">10:04 이벤트 도착 → stream-time 10:08: 해당 윈도우에서 거절</text>
</g>
</svg>
<figcaption>직접 작성한 개념도. 동일한 이벤트도 처리 시점의 stream-time에 따라 수용 여부가 달라진다.</figcaption>
</figure>

## 수용 기한과 출력 정책을 분리한다

윈도우 집계는 기본적으로 변경된 결과를 하류에 전달할 수 있다. Grace를 늘리는 것만으로 중간 결과가 숨겨지지는 않는다. 키와 윈도우별 최종 결과만 필요하면 집계 뒤에 Suppressed.untilWindowCloses를 적용하는 방식을 검토한다. 이 연산은 종료와 grace 만료까지 결과를 버퍼에 보관한다.

입력이 멈춰 stream-time이 전진하지 않으면 실제 시계가 흘러도 최종 출력이 지연될 수 있다. 따라서 이 기능을 “매시 정각에 반드시 전송”하는 스케줄러처럼 사용하면 안 된다. 실시간 화면에는 갱신 가능한 중간 집계를, 확정 알림에는 종료 후 결과를 제공하는 식으로 소비자 계약을 나눌 수 있다.

## 운영에서는 누락과 버퍼를 함께 본다

Grace는 실제 도착 지연 분포와 허용 가능한 결과 대기 시간을 기준으로 정한다. 긴 grace는 더 늦은 데이터를 수용하지만, 최종 결과 대기와 유지해야 할 상태 부담도 늘린다. 미래로 잘못 찍힌 timestamp, 장시간 입력 중단, grace 이후 도착을 각각 테스트해야 한다. 이미 닫힌 집계의 보정은 원본 보존과 별도 재집계 절차로 설계한다.

최종 결과만 출력하는 suppression에는 조기 방출하지 않는 strict buffer 설정이 필요하다. 무제한 버퍼는 메모리 고갈 위험이 있고, 용량 제한과 shutDownWhenFull을 조합하면 제한 초과 시 애플리케이션이 종료된다. 용량 증가만 반복하기보다 활성 키·윈도우 수와 버퍼 사용량을 관찰해야 한다. 최종 결과라는 의미 역시 외부 알림 시스템의 중복 전송 방지까지 보장하지는 않는다.

## 참고 자료

2026-10-26 공식 Kafka 4.3 문서와 4.3.1 API 표기를 확인했다. 신규 기능 발표가 아닌 기존 동작과 설계 기준의 해설이다.

- [Kafka Streams Core Concepts — Time](https://kafka.apache.org/43/streams/core-concepts/)
- [TimeWindows API](https://kafka.apache.org/43/javadoc/org/apache/kafka/streams/kstream/TimeWindows.html)
- [Suppressed API](https://kafka.apache.org/43/javadoc/org/apache/kafka/streams/kstream/Suppressed.html)
- [Suppressed.BufferConfig API](https://kafka.apache.org/43/javadoc/org/apache/kafka/streams/kstream/Suppressed.BufferConfig.html)

# 복권 프로모션 지급 시스템

## 개요

<div style="display: flex; align-items: flex-start; gap: 5px;">
  <div style="flex: 2;">
    <img src="image/lottery-promotion/1748115423278.png" alt="행운주사위" width="250" >
    <p style="text-align: left; font-size: 0.95em; color: #666;">2024년 여름 프로모션 홍보 이미지</p>
  </div>
  <div style="flex: 3;">
    <p style="text-align: left;">
        당근마켓 알바 플랫폼의 2024년 여름 프로모션 ‘행운 주사위' 사용자 행동 기반으로 리워드를 자동 지급하는 시스템입니다. 본 프로젝트에서는 공통 프로세스를 상속받아, 해당 프로모션 조건에 맞는 지급 흐름을 구현하고, 이벤트 기반 아키텍처 환경에서 복권을 지급하고 장애를 대응했습니다.
    </p>
  </div>
</div>

## 문제 상황

- 프로모션 페이지 최초 진입 시 기본 행운 주사위 1개 자동 지급
- 사용자가 공유한 초대 링크를 친구가 열람하면, 초대자와 친구 모두에게 각각 1개씩 추가 지급
- 보유한 복권 개수 표시, 중복 지급 방지, 지급 조건 검증 필요
- 24일(240708~240731)간 128401명 당첨 예정

## 문제 해결

```mermaid
flowchart LR
  %% Client
  subgraph CLIENT [Client]
    A[프로모션 페이지 접속]
    A2[조회 API 요청: 내 보유 주사위 몇 개인지]
  end

  %% Frontend 이벤트 수집
  subgraph FRONTEND [Frontend 이벤트 수집]
    B[프로모션 정보를 담은 이벤트 발행]
  end

  %% Agenda 처리기
  subgraph AGENDA [Node.js Agenda]
    C[프로모션 이벤트 처리기 실행]
    D[Kafka로 프로모션 이벤트 토픽 발행]
  end

  %% Kafka 소비자
  subgraph KAFKA [Kafka Consumer]
    E[EventRegister에서 프로모션 조건에 따른 유효성 검사 및 알림 발송]
    M[DB에 지급 이벤트 등록]
  end

  %% 지급 처리 시스템
  subgraph BACKEND [Backend 지급 처리]
    F[MongoDB - 지급 이력 확인 및 저장]
    G[Redis - 보유 개수 캐싱]
    H[drawInviteEventLot - 리워드 서비스 요청]
    I[Fallback - Redis에도 없고 MongoDB에도 없으면 1개 있다고 응답]
  end

  %% 리워드 서비스
  subgraph REWARD [리워드 서비스]
    J[리워드 서비스 처리]
    K[알림 서비스 - 지급 알림 요청]
    L[쿠폰함 - 기프티콘 등록]
  end

  %% 클라이언트 조회 흐름
  A --> A2
  A2 --> G
  G -->|캐시에 있음| A
  G -->|캐시에 없음| F
  F -->|지급 이력 없음| I
  I --> A

  %% 지급 이벤트 흐름
  A --> B
  B --> C
  C --> D
  D --> E
  E --> F
  E --> G
  E --> M
  E --> H
  H --> J
  J --> K
  J --> L
  K --> A
  L --> A

```

1. 최초 자동 지급 처리

- 프로모션 페이지 방문 시 Kafka를 통해 지급 이벤트 발행
- Redis에 보유 개수를 캐싱하여 빠른 응답 및 UI 노출 보장
- MongoDB에 지급 이력을 저장하여 중복 지급 방지

2. 친구 초대 기반 지급

- 사용자가 공유한 링크를 타인이 열람 시, 공유자와 열람자 모두에게 1개씩 지급
- 링크 클릭 이벤트를 수집 후 조건 검증(최초 접속 등)하여 비동기 지급 처리
- Agenda.js 기반 스케줄러로 이벤트 큐 처리

3. 리워드 당첨 처리

4. 장애 대응 로직 적용

- Redis 캐시에 보유 개수가 없을 경우 임시 1개 지급된 것처럼 응답 처리

- 지급-조회 간 타이밍 이슈에 따른 일관성 문제 해소

## 성과

- 수십만 명의 사용자 참여가 이루어진 이벤트에서 안정적인 지급 및 조회 처리
- 이벤트 기반 복권 지급 시스템 구현
- 보유 복권 개수 문의 발생 3시간 이내 대응 및 복구

## 회고

사용자의 참여 행동(접속, 공유, 클릭)에 따라 보상을 유연하게 제공해야 했던 프로젝트로, 비동기 이벤트 처리와 상태 기반 분기에 대한 이해가 요구되었습니다. 이벤트성 시스템의 실시간성, 데이터 정합성, 확장성에 대한 기술적 성장의 기회가 된 경험입니다.

## 참고 자료

- [2024년 여름 프로모션 페이지](https://daangn-jobs-campaign-2024-summer.karrotwebview.com/applicant?af_xp=custom&source_caller=ui&pid=brand_user&is_retargeting=true&shortlink=bvzgpdpt&af_adset=jobs&af_ad=summer_240704_inapp&deep_link_value=karrot%3A%2F%2Fminikarrot%2Frouter%3Fremote%3Dhttps%253A%252F%252Fdaangn-jobs-campaign-2024-summer.karrotwebview.com%252Fapplicant%26navbar%3Dtrue%26scrollable%3Dtrue&c=inapp_contents)

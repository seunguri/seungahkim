# gRPC 재시도 정책 도입 후 스트림 폭주 문제 대응

## 개요

gRPC 기반 서비스에 재시도 정책을 도입한 이후, 예상치 못한 오류가 발생했습니다.
로그와 서비스 구조, 오픈소스 이슈 분석을 통해 원인을 추정하고, 커스텀 재시도 로직을 도입하여 해결하였습니다.

## 문제상황

`@grpc/grpc-js`의 자동 재시도 기능 활성화 후, 대량 요청 시 `13 INTERNAL: RST_STREAM with code 0` 오류가 다수 발생하였습니다.
이는 Envoy가 스트림 제한을 초과할 경우 trailer 없이 스트림을 종료하면서, gRPC.js의 내부 재시도 정책이 이를 인식하지 못해 과도한 재시도가 발생했습니다.

## 개선목표

- 재시도에 의한 스트림 폭주 문제 완화
- 오류 발생 상황에 대한 원인 추정 및 대응책 도입
- gRPC 기반 서비스의 응답 안정성 확보

## 해결전략

- gRPC 프로토콜과 Envoy 동작 방식을 분석해 trailer 누락 문제를 추정
- gRPC.js 재시도 정책 대신 [`p-retry`](https://github.com/sindresorhus/p-retry)를 적용하여, 지수 백오프 기반의 재시도 로직을 구현
- Envoy trailer 누락 문제 관련 패치 확인: [Envoy PR #16219](https://github.com/envoyproxy/envoy/pull/16219),
- 관련 이슈 및 커뮤니티 자료를 참고하여, 장애 상황과 원인 가능성을 추정
  - [`grpc-js` RST_STREAM 관련 이슈](https://github.com/grpc/grpc-node/issues/2569)
  - [Istio + Envoy 환경에서의 유사 이슈](https://github.com/istio/istio/issues/50244)

## 성과

- p-retry 적용 이후, 기존에 오류가 자주 발생하던 gRPC 기반 프로세스들의 요청 성공률이 눈에 띄게 개선
- 클라이언트 재시도가 제어되며, 서비스 처리 흐름이 안정화
- 운영 중이던 오류성 API 호출에 대한 내부 대응 부담이 감소

## 사용기술 및 도구

- Node.js, gRPC.js (`@grpc/grpc-js` v1.11.1)
- p-retry v4.0.0
- Istio / Envoy
- Slack, GitHub Issues, 사내 위키 등

## 개인 기여

- 클라이언트 로그 기반의 오류 패턴 분석
- gRPC 및 Envoy의 동작 방식 리서치 및 원인 추정
- 재시도 정책 설계 및 적용
- 대응 내용 사내 공유 및 장기 개선 제안

## 회고

gRPC 재시도 정책이 Envoy의 max concurrent streams 제한과 충돌하면서 발생한 문제를 통해, 인프라 설정이 애플리케이션 안정성에 어떤 영향을 주는지 경험했습니다.
오픈소스 이슈와 커뮤니티 자료를 적극적으로 활용하는 경험을 쌓고 관련 내용을 팀 내부에 정리해 공유했습니다.

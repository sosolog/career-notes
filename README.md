# Career Notes

백엔드 개발을 중심으로,
서비스 운영에 필요한 인프라 구성부터 배포 자동화까지
전체 라이프사이클을 경험해온 엔지니어입니다.

작은 조직 환경에서 한정된 리소스 안에서
실용적인 해결책을 선택하고 이를 직접 구현하는 역할을 해왔으며,
이 저장소는 그 과정에서의 기술적 판단과 trade-off를 정리하기 위한 공간입니다.

---

## 📂 기술 판단 기록

### 📡 IoT 텔레메트리 처리 지연 개선 (30분 → 1.5분)

MQTT 기반 텔레메트리 저장 지연이 최대 30분까지 누적되는 문제를 해결했습니다.
병렬 처리 시도(Deadlock 발생), MyBatis 배치 모드 부분 적용을 거쳐
상태 계산 로직을 Procedure로 이관하여 DB roundtrip을 줄이는 방향으로 해결했습니다.

→ [상세 기록 보기](docs/telemetry-improvement.md)

---

### 🔍 Prometheus 기반 통합 모니터링 시스템 구축

공식 Exporter가 없는 Tibero DB 환경에서 Custom Exporter를 직접 개발했습니다.
시계열/비시계열 데이터 특성에 따라 수집 구조를 분리하고,
프론트엔드 다중 API 호출 문제를 미들웨어 통합 수집 + WebSocket push 방식으로 개선했습니다.

→ [상세 기록 보기](docs/prometheus-custom-exporter.md)

---

### 📬 메일 / 메시지 중계 서비스 아키텍처 설계

프로젝트별로 분산된 발송 로직을 중앙화했습니다.
Sender / Scheduler / Agent 구조를 설계하면서
글로벌 공유 자원과 시스템별 자원을 어떻게 분리할지를 고민한 과정을 기록했습니다.

→ [상세 기록 보기](docs/message-relay-architecture.md)

---

### 🧩 Spring Boot 기반 공통 설정 및 레거시 연동 모듈 개발

JPA/MyBatis, 다중 DB 설정을 YAML 선언만으로 자동 구성되도록 설계했습니다.
BeanPostProcessor의 한계를 확인하고 BeanDefinitionRegistryPostProcessor로 전환한 과정,
Nexacro 기반 레거시 시스템을 Spring Boot 환경에 연동한 과정을 기록했습니다.

→ [상세 기록 보기](docs/common-modules.md)

---

### 🚀 단일 서버 Docker 환경에서의 Blue/Green 무중단 배포

Kubernetes 없이, 단일 서버 + Nginx + Docker 환경에서
포트 스위칭 기반 Blue/Green 배포 구조를 직접 구현했습니다.
운영 중 DRM 라이브러리 크래시 문제를 겪고 Python 헬스체크 프로세스를 추가한 과정도 담았습니다.

→ [상세 기록 보기](docs/blue-green-deployment.md)

---

## 🔧 사용한 기술 범위

| 영역 | 기술 |
|------|------|
| Backend | Java, Spring Boot, JPA, MyBatis, Spring Batch |
| Monitoring | Prometheus, Custom Exporter, WebSocket |
| Deployment | Docker, GitHub Actions, CodeDeploy, Nginx |
| Messaging | MQTT, Kafka (구조 검토) |
| Legacy | Nexacro 연동 |

---

## ✍️ 이 저장소에 대하여

기술을 과시하기보다는,
제한된 환경 속에서 어떤 선택을 했고 그 선택이 어떤 결과로 이어졌는지를 기록하는 것을 목표로 합니다.

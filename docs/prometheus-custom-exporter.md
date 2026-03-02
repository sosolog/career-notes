# Prometheus 기반 통합 모니터링 시스템 구축

## 배경

도입사로부터 아래 세 가지 모니터링 요구사항이 있었습니다.

- 서버 5대의 상태 (CPU, RAM, Disk, Network)
- DB 상태 (Active Session, 락, 현재 실행 쿼리 등)
- 배치 작업 상태

---

## 서버 모니터링 — 기존 Exporter 활용

서버 상태는 **Node Exporter**와 **cAdvisor**를 도입하여 Prometheus로 수집했습니다.
추가로 DB 서버에서 Tibero 프로세스의 리소스 사용량을 추적하기 위해 **Process Exporter**도 도입했습니다.

**문제: Process Exporter 부하**

운영 환경에서 Tibero를 추적하도록 설정하자, `tb_listener` 같은 하위 프로세스들이 너무 많아
메트릭 수집에 부하가 걸렸습니다.
Process Exporter 자체도 다른 영역에서 부하가 높아, 결국 **Process Exporter는 걷어냈습니다.**

---

## DB 모니터링 — Tibero Custom Exporter 개발

DB 상태 수집은 보통 DB Exporter를 사용하지만, **Tibero는 공식 Exporter가 없었습니다.**

**Oracle Exporter 검토 및 포기**

Tibero가 Oracle 기반이라 Oracle Exporter를 검토했습니다.
Exporter 내부를 확인해보니 주기적으로 특정 쿼리를 날리는 구조였는데,
쿼리에서 사용하는 테이블이 Tibero에는 존재하지 않았습니다.
Oracle Exporter는 사용할 수 없었습니다.

**Tibero Custom Exporter 개발**

Tibero 공식 문서에 모니터링용 쿼리 가이드가 있었습니다.
Oracle Exporter의 구조(주기적으로 쿼리를 날려 메트릭 수집)를 참고하고,
Tibero 공식 문서의 쿼리를 기반으로 **Java / Spring Boot로 Custom Exporter를 개발**했습니다.

Go가 Exporter 개발의 일반적인 선택이지만, 빠른 개발이 필요했고
팀 내 주 언어가 Java였기 때문에 Java를 선택했습니다.

수집 메트릭은 Oracle Exporter 제공 항목과 Tibero 공식 문서 모니터링 항목을 함께 참고하여 선정했습니다.

---

## 비시계열 데이터 문제 — 미들웨어 분리

모니터링 항목 중 **현재 락 정보, 현재 실행 중인 쿼리 목록** 같은 데이터는 성격이 달랐습니다.

이런 데이터를 Prometheus로 수집하면 메트릭 개수가 폭발적으로 늘어납니다.
Prometheus는 시계열 데이터에 최적화된 구조라, 리스트 형태의 비시계열 데이터에는 맞지 않습니다.

**해결: 미들웨어에서 별도 수집**

비시계열 데이터는 Prometheus Exporter가 아닌 **미들웨어에서 직접 SQL로 수집**하도록 분리했습니다.

```
시계열 데이터 (CPU, Session 수 등) → Custom Exporter → Prometheus
비시계열 데이터 (락, 실행 쿼리 등) → 미들웨어에서 직접 SQL 수집
```

---

## 화면 부하 문제 — WebSocket Push 방식으로 통합

원래 설계는 프론트엔드에서 Prometheus에 직접 쿼리하는 방식이었습니다.
그런데 확인해야 할 메트릭 수가 많아, 화면에서 API를 일일이 호출하면 요청 수가 너무 많아졌습니다.

**해결: 미들웨어에서 통합 수집 후 WebSocket Push**

Prometheus 쿼리도 미들웨어로 이동하여,
시계열 데이터(Prometheus 쿼리 결과)와 비시계열 데이터(직접 SQL)를 미들웨어에서 통합한 뒤
**WebSocket으로 한 번에 Push**하는 방식으로 변경했습니다.

```
프론트엔드 ← WebSocket Push ← 미들웨어
                                ├── Prometheus 쿼리 (시계열)
                                └── 직접 SQL (비시계열: 락, 실행 쿼리)
```

프론트엔드 입장에서는 WebSocket 하나만 구독하면 되고,
Prometheus 서버 정보가 프론트엔드에 직접 노출되지 않는 구조도 확보됐습니다.

---

## 최종 구조 요약

| 항목 | 수집 방식 |
|------|-----------|
| 서버 CPU / RAM / Disk / Network | Node Exporter → Prometheus |
| 컨테이너 상태 | cAdvisor → Prometheus |
| Tibero DB 상태 (시계열) | Custom Exporter → Prometheus |
| Tibero 락 / 실행 쿼리 (비시계열) | 미들웨어 직접 SQL |
| 배치 상태 | 미들웨어 직접 수집 |
| 프론트엔드 전달 | 미들웨어 → WebSocket Push |

---

## 한계 및 트레이드오프

- Process Exporter를 걷어내면서 Tibero 프로세스 단위 리소스 추적은 포기했습니다. DB 서버 전체 리소스와 Tibero 메트릭으로 간접 파악하는 수준입니다.
- 미들웨어가 Prometheus 쿼리와 직접 SQL을 모두 담당하면서 역할이 커졌습니다. 모니터링 항목이 늘어날수록 미들웨어 복잡도가 높아질 수 있습니다.
- 장애 탐지보다는 운영 상태를 확인하는 용도로 활용됐습니다. 알림 체계까지는 구축하지 못했습니다.

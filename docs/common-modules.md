# Spring Boot 기반 공통 설정 및 레거시 연동 모듈 개발

## 배경

여러 프로젝트를 진행하면서 반복되는 문제가 두 가지 있었습니다.

- **초기 설정 중복**: JPA/MyBatis, 다중 DB 설정을 프로젝트마다 새로 구성
- **레거시 연동 부담**: Nexacro 기반 시스템을 Spring Boot에 연동할 때마다 동일한 작업 반복

이를 공통 모듈로 만들어 한 번만 구현하고 여러 프로젝트에서 재사용하는 것이 목표였습니다.

---

## YAML 기반 자동 구성 — BeanDefinitionRegistryPostProcessor

### 목표

YAML 설정만으로 DataSource, SqlSessionFactory, EntityManagerFactory 등
필요한 빈이 자동으로 등록되도록 구성하는 것이었습니다.

### 시도 1 — BeanPostProcessor

처음에는 `BeanPostProcessor`를 사용하려 했습니다.
YAML에서 읽은 DB 정보를 빈으로 등록하고,
이를 기반으로 DataSource 등 필요한 빈을 만드는 방식이었습니다.

**문제: @Service 같은 컴포넌트에서 주입이 안 됨**

`BeanPostProcessor`는 빈이 이미 생성된 이후에 동작하는 확장 포인트입니다.
이 시점에 새로운 빈을 등록해도, `@Service` 같은 컴포넌트에서 정상적으로 주입받으려면
`@Lazy`를 붙여야 하는 문제가 생겼습니다.
사용하는 쪽에서 매번 `@Lazy`를 신경 써야 한다면 공통 모듈의 의미가 없었습니다.

### 시도 2 — BeanDefinitionRegistryPostProcessor

Spring 빈 생명주기를 다시 확인했습니다.

```
BeanDefinition 등록
       ↓
BeanDefinitionRegistryPostProcessor  ← 이 시점에 개입
       ↓
BeanPostProcessor
       ↓
빈 생성 완료
```

`BeanDefinitionRegistryPostProcessor`는 빈이 생성되기 전,
BeanDefinition이 등록되는 단계에 개입할 수 있습니다.
이 시점에 DataSource, SqlSessionFactory 등의 BeanDefinition을 미리 등록해두면
이후 `@Service`에서 `@Lazy` 없이 자연스럽게 주입받을 수 있었습니다.

**결과**

사용하는 쪽에서는 YAML 설정만 작성하면 되고,
`@Lazy` 없이 정상적으로 빈을 주입받을 수 있게 됐습니다.

---

## Nexacro 레거시 연동 — Java 17 마이그레이션 및 어노테이션 기반 분기

### 배경

Nexacro는 데이터를 JSON이 아닌 XML 기반의 자체 포맷으로 주고받습니다.
기존에 Java 8 기반 오픈소스 연동 라이브러리가 있었고,
이를 Java 17 환경에 맞게 수정하여 적용했습니다.

### 문제 — REST 엔드포인트와 Nexacro 엔드포인트 구분

하나의 Spring Boot 프로젝트에서 일반 REST API와 Nexacro 엔드포인트가 공존해야 했습니다.
둘은 데이터 구조가 달라 파싱과 응답 변환을 다르게 처리해야 했습니다.

**요청 파싱**
Nexacro 데이터 타입 클래스가 메서드 파라미터로 들어오면
Nexacro XML 포맷으로 파싱해야 하고,
일반 클래스가 들어오면 기존 방식대로 처리해야 했습니다.

**응답 변환**
응답도 마찬가지로, Nexacro 엔드포인트는 Nexacro 포맷으로,
일반 엔드포인트는 JSON으로 내려줘야 했습니다.

### 해결 — HandlerArgumentResolver + @NexacroBody 어노테이션

**요청 파싱 — HandlerArgumentResolver**

`HandlerArgumentResolver`에서 파라미터 타입이 `DataSetList`, `VariableList` 같은
Nexacro 데이터 타입인지 확인하여 Nexacro XML 포맷으로 파싱하도록 구현했습니다.
일반 파라미터 타입이면 기존 방식대로 처리됩니다.

**응답 변환 — @NexacroBody**

`@NexacroBody`는 메서드 또는 클래스 단에 붙이는 어노테이션입니다.
이 어노테이션이 붙어 있으면 응답을 Nexacro 포맷으로 변환하고,
없으면 일반 JSON 응답으로 처리됩니다.

```java
// Nexacro 엔드포인트 — 클래스 단 혹은 메소드에 붙이면 전체 적용

@Controller
public class NexacroController {
    @NexacroBody
    @PostMapping("/submit")
    public NexacroResult submit(DataSetList dataSetList, VariableList variableList) { ... }
}

// 일반 REST 엔드포인트 — 기존 방식 그대로
@RestController
public class ApiController {
    @PostMapping("/api/data")
    public ResponseEntity<?> getData(@RequestBody SomeDto dto) { ... }
}
```

---

## 한계 및 트레이드오프

- `BeanDefinitionRegistryPostProcessor`는 Spring 내부 메커니즘을 직접 다루는 만큼,
  Spring 버전 업그레이드 시 동작이 달라질 수 있는 위험이 있습니다.
- Nexacro 연동 라이브러리는 오픈소스를 기반으로 수정한 것이라,
  Nexacro 버전이 바뀌면 재검토가 필요합니다.
- 공통 모듈이 여러 프로젝트에서 사용되기 때문에,
  변경 시 영향 범위가 넓어 신중하게 수정해야 합니다.

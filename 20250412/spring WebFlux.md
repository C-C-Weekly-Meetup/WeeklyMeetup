# spring webflux - 회사 프로젝트 덕분에 제대로 파봤다

최근 회사에서 맡게 된 프로젝트가 전부 **Reactive 방식**으로 작성되어 있었다.

처음엔 그냥 익숙한 방식대로 코드를 따라가려 했는데, `Mono`, `Flux`, `flatMap` 이런 낯선 애들이 자꾸 튀어나오길래… "이건 안 보고는 못 지나가겠다" 싶어서 본격적으로 정리하게 되었다.

이 글은 내가 그때 조사하면서 정리한 내용을 바탕으로,

**“Spring Reactive가 뭔지, 언제 쓰는지, 왜 쓰는지”** 가볍게 정리한 기록이다.

---

## spring webflux 뭐냐면

한마디로 말하면 **비동기로 데이터 흐름을 처리하는 방식**이다.

기존에는 함수가 리턴되면 그 값이 바로 나왔지만, spring webflux 에선 "나중에 값이 올 거야"라는 걸 미리 알려주는 구조다.

여기서 핵심은 이 두 개:

- `Mono`: 1개 또는 0개의 데이터 (ex. 단일 조회)
- `Flux`: 여러 개의 데이터 (ex. 리스트, 스트리밍)

---

## 왜 굳이 이걸 써?

Spring MVC는 요청이 오면 스레드를 하나 점유하고, 응답이 끝날 때까지 붙잡고 있는 구조다. 이건 트래픽이 적을 땐 괜찮은데, 많아지면 스레드 부족으로 병목이 생긴다.

반면에 Reactive는 **논블로킹 방식**이라, 스레드를 오래 잡고 있지 않는다.

외부 API나 DB I/O 같은 느린 작업도 가볍게 넘길 수 있어서, **동시 요청이 많은 시스템**에서 효과적이다.

---

## WebFlux는 뭐가 다른가?

Spring WebFlux는 Spring에서 지원하는 **Reactive Web 프레임워크**다.

기존 MVC랑 겉보기엔 비슷하지만, 내부는 완전히 다르다.

```java
java
복사편집
@GetMapping("/users/{id}")
public Mono<User> getUser(@PathVariable String id) {
    return userService.findById(id);
}

```

위처럼 `User` 대신 `Mono<User>`로 리턴한다.

이렇게 하면 "나중에 유저 정보 줄게~" 하고 응답을 미뤄두는 구조가 된다.

---

## 언제 쓰면 좋을까?

- 외부 API 호출이 많은 서비스
- 동시 접속자 수가 많은 경우
- 서비스 간 연동이 잦은 MSA 구조

**단, CPU 연산이 많은 서비스는 오히려 MVC가 더 나을 수도 있다.**

---

### Mono 요약 정리

- `Mono`는 0 또는 1개의 데이터를 비동기적으로 감싸는 타입
- `just()`: 값을 감싸서 Mono로 만듦
- `flatMap()`: 비동기 로직 이어가기
- `error()`: 예외를 Mono로 전달
- `defaultIfEmpty()`: 값이 없을 때 기본값 지정

## 써보면서 느낀 팁

- **블로킹 API는 피하는 게 좋다`(Mono.flatMap().block()`)**
    
    예: JPA, RestTemplate → 대신 R2DBC, WebClient 쓰는 게 자연스럽다.
    
- **ThreadLocal 쓰지 말자**
    
    Reactive는 스레드가 왔다갔다 하니까 `Reactor Context`로 처리하는 게 훨씬 안정적이다.
    
- **테스트도 방식이 다르다**
    
    그냥 assertEquals로 끝나지 않는다. `StepVerifier` 같은 도구로 스트림 흐름을 체크해주는 게 좋다.
    

### 테스트 코드

- `StepVerifier.create(mono)`로 Mono 테스트 흐름 시작
- `.assertNext()`로 emit된 값 검증
- `.verifyComplete()`로 정상 완료 여부 확인

예제처럼 응답 코드, 파싱 결과, 예외 발생 여부 등을 체크할 때 유용하다.

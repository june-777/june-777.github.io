---
title: "스프링 ApplicationEventPublisher와 EventListener"
date: 2024-11-21 12:00:00
categories:
- 스프링
tags:
- 스프링
- 트랜잭션 전파
- 비동기 처리
---

스프링 이벤트 발급자와 이벤트 리스너를 사용할 때, 트랜잭션 전파 관점에서 살펴봅니다. 추가로 비동기로 분리할 때의 주의점도 살펴봅니다.

## 트랜잭션 전파 관점

### @EventListener

`ApplicationEventPublisher`를 의존하여 이벤트를 발행하는 측을 이벤트 발행자, `@EventListener`에서 수신하여 실행하는 측을 이벤트 수신자라고 이야기하겠습니다.

```java
// 이벤트 발행자
@Service
@Transactional
@RequiredArgsConstructor
class RecruitmentService {
  
    private final ApplicationEventPublisher applicationEventPublisher;
  
    public void bizMethod() {
    // ...
    applicationEventPublisher.publishEvent(event);
  }
}
```

```java
// 이벤트 수신자
@Component
public class ApplicationEventListener {
    private final NotificationService notificationService;
	
    @EventListener
    public void onApplicationEvent(RecruitmentEvent event) {
    // 이벤트를 수신하여 실행시킬 로직
    notificationService.notify(event);
  } 
}
```

<br>

이벤트 발행자가 스프링 트랜잭션에 의해 관리되고 있다면, 기본적으로는 이벤트 수신자도 **기존의 트랜잭션을 그대로 사용**하게 됩니다. 하나의 스레드에서 동기적으로 실행되고 있기 때문입니다. 만약 수신측에서 전파 옵션이 `REQUIRED`인 트랜잭션을 사용하고 있는 경우에도 동일합니다.

아래의 그림에서 `NotificationService`에서 예외가 발생하여 롤백되면, `RecruitmenetService`도 함께 롤백됩니다. 다시말해, **이벤트 수신측에서 롤백이 발생하면 이벤트 발행측도 함께 롤백**됩니다.

<img src="https://github.com/user-attachments/assets/263deec5-7e4b-4fc2-b436-29e56ddfe50c">

<br>

### REQUIRES_NEW

비즈니스 요구사항에 맞게 이벤트 발행측과 수신측의 트랜잭션 분리 여부를 고민해볼 필요가 있습니다. 만약 트랜잭션 분리가 필요하다면, 새로운 트랜잭션을 생성하는 전파 옵션인 `REQUIRES_NEW`를 고려해볼 수 있겠습니다.

<img src="https://github.com/user-attachments/assets/714752be-7a4b-48b6-b740-a305ca597ff4">

<br>

이 때 이벤트 발행측에서는 수신측에서 발생할 수 있는 **예외를 핸들링하여 정상처리**해주어야 하는 것을 주의해야합니다. 발행측과 수신측은 여전히 하나의 스레드에서 동작하고 있기 때문에, 수신측에서 발생한 **예외가 발행측으로 전파**되어, 발행측은 실제적으로 성공했음에도 `@Transactional AOP Proxy`가 실패로 인식하여 함께 롤백하기 때문입니다.

`발행측에서 수신측의 예외를 핸들링하지 않았을 경우`

<img src="https://github.com/user-attachments/assets/2d8a2d60-ea34-4d13-a7d7-3471ae367959">

```java
// 발행측 (RecruitmentService)에서 수신측의 예외를 핸들링하지 않았을 경우
@Test
@DisplayName("트랜잭션 전파 옵션을 requires_new로 설정하면, "
        + "발행자 / 수신자의 트랜잭션이 분리되지만, "
        + "발행자 측에서 예외 핸들링하지 않으면 예외가 전파되어 함께 롤백된다.")
void eventListenerWithPropagationRequiresNewException() {
    // given:
    String invalidRecruitmentTitle = "invalid";

    // when: 알림 서비스에서 예외 발생, 모집공고 서비스에서 예외 핸들링 X
    assertThatThrownBy(() -> recruitmentService.create(user, invalidRecruitmentTitle))
            .isInstanceOf(NoSuchElementException.class);

    // then:
    assertThat(recruitmentRepository.findAll()).isEmpty();	// 함께 롤백
    assertThat(notificationRepository.findAll()).isEmpty();	// 롤백
}
```

<br>

`발행측에서 수신측의 예외를 핸들링했을 경우`

<img src="https://github.com/user-attachments/assets/dd5f7c57-76c8-4682-ae45-30d2ea3a48ed">

```java
@Service
@Transactional
@RequiredArgsConstructor
@Slf4j
public class RecruitmentService {

    private final RecruitmentRepository recruitmentRepository;
    private final ApplicationEventPublisher applicationEventPublisher;

    public Recruitment create(final User user, final String recruitmentTitle) {
    // ...
    // 예외 핸들링 및 정상 로직 복구
        try {
            applicationEventPublisher.publishEvent(new RecruitmentEvent(recruitment.getId()));
        } catch (Exception e) {
            log.info("알림에서 예외 발생 및 예외를 정상 복구함-{}", e.getMessage());
        }
    // ...
        return recruitment;
    }
}
```

```java
// 예외를 핸들링했을 경우
@Test
@DisplayName("트랜잭션 전파 옵션을 requires_new로 설정하면, 발행자 / 수신자의 트랜잭션이 분리된다.")
void eventListenerWithPropagationRequiresNew() {
    // given:
    String invalidRecruitmentTitle = "invalid";

    // when: 알림 서비스에서 예외 발생, 모집공고 서비스에서 예외 정상 처리
    recruitmentService.create(user, invalidRecruitmentTitle);

    // then: 모집공고는 커밋
    assertThat(recruitmentRepository.findAll()).isNotEmpty();	// 모집공고 서비스는 성공하여 커밋
    assertThat(notificationRepository.findAll()).isEmpty();		// 알림서비스는 실패하여 롤백
}
```

하지만 이와 같은 방법은 발행자가 **수신자의 예외를 모두 알아야하는하는 문제**가 있습니다. 이벤트를 기반으로 분리하는 것은 클래스 간의 강결합을 약결합으로 분리하여, 변경의 파급효과를 최소화하는 것이 장점 중 하나인데, 이 장점이 퇴색될 수 있습니다. `@EventListner`에서 예외를 핸들링하는 방법도 있지만, 근본적인 문제를 해결하지는 못한다고 생각합니다.

<br>

### @TransactionalEventListener

알림 이벤트 발행측과 수신측의 트랜잭션 분리를 위해 `REQUIRES_NEW`를 사용했습니다. 하지만 발행측에서 수신측의 예외를 알아야하는 문제가 생겼습니다. 이 문제를 해결하기 위해 발행자의 트랜잭션이 완전히 종료된 후 수신자를 시작하는 방법을 생각해볼 수 있겠습니다. 이 때 고려할만한 방법이 `@TransactionalEventListener`입니다.

`TransactionalEventListener`는 트랜잭션의 특정 단계에서 이벤트 리스너를 동작할 수 있도록 하는 애노테이션입니다.

- AFTER_COMMIT(기본값): 트랜잭션이 커밋된 직후

- AFTER_ROLLBACK: 트랜잭션이 롤백된 직후

- AFTER_COMPLETION: 트랜잭션이 완료된 후 (커밋 또는 롤백 여부에 상관없음)

⭐️ 여기서 주의해야할 것은 **수신측에서 발생하는 커밋은 무시한다는 것**입니다. 이벤트 리스너가 **호출된 시점엔 기존의 트랜잭션이 커밋 또는 롤백되어 이미 종료**되었기 때문입니다.

> @TransactionalEventListner Docs:
>
> WARNING: if the TransactionPhase is set to AFTER_COMMIT (the default), AFTER_ROLLBACK, or AFTER_COMPLETION, the transaction will have been committed or rolled back already, but the transactional resources might still be active and accessible. As a consequence, any data access code triggered at this point will still "participate" in the original transaction, but changes will not be committed to the transactional resource. See TransactionSynchronization. afterCompletion(int) for details.

```java
/**
 * Listener: @TransactionalEventListener
 * NotificationService txPropagation: REQUIRED
 */
@Test
@DisplayName("수신측의 트랜잭션 전파 옵션이 REQUIRED 인 경우 기존의 트랜잭션과 합쳐지고, "
        + "수신측 실행 시점 (after_commit)엔 이미 커밋되었기 때문에 수신측의 커밋은 무시된다.")
void test() {
    // given, when: 모집공고 서비스 성공, 알림 서비스 성공
    recruitment = recruitmentService.create(member, recruitmentTitle);

    // then
    assertThat(recruitmentRepository.findAll()).isNotEmpty();	// 모집공고는 커밋 성공
    assertThat(notificationRepository.findAll()).isEmpty();		// 알림은 커밋 무시
}
```

<br>

따라서 해당 애노테이션을 사용해야 할 경우, **이벤트 수신측의 트랜잭션을 `REQUIRES_NEW`로 새로 생성**하는 것을 고려하면 되겠습니다.

<img src="https://github.com/user-attachments/assets/3091a775-405a-432d-9b4b-27c43fac31cd">

```java
/**
 * Listener: @TransactionalEventListener
 * NotificationService txPropagation: REQUIRES_NEW
 */
@Test
@DisplayName("수신측의 트랜잭션 전파 옵션이 REQUIRES_NEW 인 경우, 새로운 트랜잭션이 생성되었으므로 커밋된다.")
void test2() {
    // given, when: 정상 모집공고 저장 성공, 알림 이벤트 발행 성공
    recruitment = recruitmentService.create(member, recruitmentTitle);

    // then
    assertThat(recruitmentRepository.findAll()).isNotEmpty();	// 모집공고 커밋 성공
    assertThat(notificationRepository.findAll()).isNotEmpty();		// 알림 커밋 성공
}
```

<br>

### 비동기 처리

이벤트 발행자와 수신자의 트랜잭션 분리를 위해 비동기 처리를 고려해볼 수 있겠습니다. 앞서 살펴본 방법은 동기적으로 실행하기 때문에, 이벤트 수신자의 작업이 오래걸리는 경우 **HTTP 트랜잭션의 응답 시간** 측면에서 성능이 좋지 않을 수 있습니다. 아래 그림을 보면, 모집공고 생성 관련 로직은 짧은 시간에 끝나지만, 모집공고 알림 관련 로직의 시간으로 인해 HTTP 응답시간이 오래걸릴 것입니다.

<img src="https://github.com/user-attachments/assets/e9160e43-f425-4173-a38f-3f56e6f16feb" width=500>

<br>

하지만 이벤트 수신자의 로직을 비동기로 분리한다면 아래와 같은 그림이 될 것입니다.

<img src="https://github.com/user-attachments/assets/e24ebdff-1f10-42ff-829f-96a9592351f9" width=500>

<br>

이벤트 리스너를 비동기로 처리하기 위해선, `@EnableAsync`비동기 환경설정을 추가하고, 이벤트 리스너에  `@Async` 애노테이션을 추가하면 됩니다. 비동기 스레드 풀과 관련한 더 다양한 내용들이 많지만, 이번 포스팅에선 다루지 않겠습니다.

```java
@Configuration
@EnableAsync
public class AsyncConfig {
}
```

```java
@Component
@Slf4j
@RequiredArgsConstructor
public class ApplicationEventListener {
    private final NotificationService notificationService;

    @EventListener
    @Async
    public void onApplicationEvent(RecruitmentEvent event) {
        boolean txActive = TransactionSynchronizationManager.isActualTransactionActive();
        log.info("[ApplicationEventListener] event! {} received, txActive - {}", event, txActive);

        notificationService.notifyWithNewTx(event);

        log.info("[ApplicationEventListener] event finished, txActive - {}", txActive);
    }

}
```

<br>

`@Async`와 `@EventListener`를 함께 사용할 때 주의해야할 점이 있습니다. 비동기는 **작업 실행 순서를 보장하지 않는 것**을 의미합니다. 즉, 이벤트 발행측에서 관리하고 있는 **트랜잭션 자원이 커밋되기 전에, 수신측에서 동일한 자원을 접근**하려 할 때 문제가 발생합니다.

<img src="https://github.com/user-attachments/assets/06f6cb96-dd15-42d6-955d-6a0700d1d5da">

```java
/**
 * Listener: @EventListener
 * NotificationService txPropagation: REQUIRES_NEW
 * 일부 테스트가 실패해야 정상
 */
@RepeatedTest(50)
@DisplayName("비동기는 작업 실행 순서를 보장하지 않기 때문에, @EventListener 를 사용하면"
                + "이벤트 발행측에서 트랜잭션 자원이 커밋되기 전에"
                + "이벤트 수신측에서 접근할 수 있어 문제가 발생할 수 있다.")
void test() {
    // given, when: 정상 모집공고 저장 성공, 알림 이벤트는 일부 테스트에서 NoSuchElementException 발생
    recruitment = recruitmentService.create(member, recruitmentTitle);

    // then
    assertThat(recruitmentRepository.findAll()).isNotEmpty(); // 모집공고 커밋 성공

    // 비동기 작업 추적
    await().atMost(1, TimeUnit.SECONDS)
            .untilAsserted(() ->
                    assertThat(notificationRepository.findAll()).isNotEmpty()	// 일부 테스트는 실패한다.
            );
}
```

<br>

이런 경우에는 앞서 살펴본 `@TransactionalEventListener`와 `@Async`를 함께 사용하면 되겠습니다.

```java
@Component
@Slf4j
@RequiredArgsConstructor
public class ApplicationEventListener {
    private final NotificationService notificationService;

    @TransactionalEventListener
    @Async
    public void onApplicationEvent(RecruitmentEvent event) {
        boolean txActive = TransactionSynchronizationManager.isActualTransactionActive();
        log.info("[ApplicationEventListener] event! {} received, txActive - {}", event, txActive);

        notificationService.notifyWithNewTx(event);

        log.info("[ApplicationEventListener] event finished, txActive - {}", txActive);
    }

}
```

```java
/**
 * Listener: @TransactionalEventListener
 * NotificationService txPropagation: REQUIRES_NEW
 */
@RepeatedTest(50)
@DisplayName("@TransactionalEventListener 를 사용하면 "
                + "이벤트 발행측의 트랜잭션이 종료되고 비동기 로직을 호출하기 때문에"
                + "커밋 타이밍으로 인한 문제가 발생하지 않는다.")
void test2() {
    // given, when: 정상 모집공고 저장 성공, 알림 서비스도 성공
    recruitment = recruitmentService.create(member, recruitmentTitle);

    // then
    assertThat(recruitmentRepository.findAll()).isNotEmpty(); // 모집공고 커밋 성공

    // 비동기 작업 추적
    await().atMost(1, TimeUnit.SECONDS)
            .untilAsserted(() ->
					assertThat(notificationRepository.findAll()).isNotEmpty()
			);
}
```


---

#### Reference:
- Spring Docs
- Blog: https://innu3368.tistory.com/273


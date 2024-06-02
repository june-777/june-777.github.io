---
layout: single
title:  "알림 기능을 Testable Code로 리팩터링"
toc: true
toc_sticky: true
---
> 알림 기능을 구현하며 테스트 용이한 코드로 리팩터링한 사례를 공유하고자 합니다.  
> 다소 정답이 없는 설계적 영역이므로, 주관적 견해인 점을 참고하시면 좋을 것 같습니다.  
> 편의를 위해 비격식체를 사용합니다.


## 테스트 어려운 코드
테스트하기 어려운 코드는 여러가지 이유가 있다. 특히 **내부 구현에 `DB I/O`가 포함되어 있는 경우**, 테스트가 어렵다고 느껴졌다. 아래의 `NotificationService` 코드 예시를 살펴보자.

`NotificationService` 객체는 알림 관련 비즈니스 요구사항이 응집되어 있는 객체이다. 내부 구현의 전반적 흐름은 `1. 알림 대상자 조회`, `2. 알림 저장`, `3. 실시간 알림 전송 요청` 으로 진행된다.  

```java
@Service
@RequiredArgsConstructor
public class NotificationService {

	// for notifier find
	private final UserRepositoryImpl userRepository;
	private final ParticipantRepositoryImpl participantRepository;
	private final ReviewRepositoryImpl reviewRepository;

	// for notification save
	private final StudyNotificationRepositoryImpl studyNotificationRepository;
	private final RecruitmentNotificationRepositoryImpl recruitmentNotificationRepository;
	private final ReviewNotificationRepositoryImpl reviewNotificationRepository;

	// for notification server sent events
	private final SseEmitters sseEmitters;

	@Transactional
	public void recruitmentNotice(final Recruitment recruitment) {
		// 알림 대상자 조회
		final RecruitmentNotifierCondition recruitmentNotifierCondition = new RecruitmentNotifierCondition(
				recruitment.getCategory(),
				recruitment.getPositions(),
				recruitment.getStacks());
		final List<User> recruitmentNotifiers = userRepository.findRecruitmentNotifiers(recruitmentNotifierCondition);

		// 알림 저장
		final List<RecruitmentNotification> recruitmentNotifications = recruitmentNotificationRepository.saveAll(
				recruitmentNotifiers
						.stream()
						.map(notifier -> RecruitmentNotification.of(
								RECRUITMENT, LocalDateTime.now(), recruitment, notifier))
						.toList()
		);

		// 실시간 알림 전송 요청
		recruitmentNotifications.forEach(recruitmentNotification -> {
			final User notifier = recruitmentNotification.getNotifier();
			final NotificationResponse notificationResponse = new NotificationResponse(recruitmentNotification);
			sseEmitters.sendNotification(notifier, notificationResponse);
		});
	}

    // something public method ...
}
```

이 코드는 테스트하기 어려운 코드이다. `1. 알림 대상자 조회`, `2. 알림 저장`을 위해 `DB I/O`를 수행하고 있으며, 해당 기능들은 내부 구현이기 때문에 정상 동작하는지 테스트하기 마땅치 않다.

<br><br>

## 무엇을 테스트해야 하는가
`Testable Code`로 리팩터링하기 전에, 무엇을 테스트해야 하는지 고민할 필요가 있다.

만약 이런 코드가 있다고 해보자.
```java
public class SomethingService {

    private final SomethingRepo somethingRepo;

    public bizMethod() {
        // 간단한 조회 로직
        Domain domain =somethingRepo.find();
        // 조회 결과를 바탕으로 복잡한 핵심 비즈니스 로직
    }
}
```

이럴 땐 조회 결과에 대한 값을 `Stubbing`하고 핵심 요구사항만 테스트하는 것을 고려해볼 수 있다. 조회 로직이 비교적 간단하며, 핵심 비즈니스 로직이 아니기 때문이다.

하지만 위에서 언급한 `NotificationService`는 조금 다른 경우이다. **`알림 유형`에 부합하는 `알림 대상자`를 선정하는 것은 알림 기능의 핵심 비즈니스 요구사항**이다. Ludo 스터디 매칭 플랫폼 서비스는 10가지 `알림 유형`이 존재하는데, 그 중 몇 가지를 예시로 살펴보자.  

[`상호 리뷰완료 알림`에 대한 알림 대상자]  
동일한 스터디에서 서로에게 리뷰를 진행한 두 사용자

[`관심 모집공고 알림`에 대한 알림 대상자]  
`스터디 카테고리 키워드`, `모집 포지션 키워드`, `모집 기술스택 키워드` 각각 최소 하나씩 모두 일치하는 사용자

`관심 모집공고 알림`유형에 대한 알림 대상자 조회로직은 다음과 같다. 딱봐도 복잡해보인다.

```java
@Repository
@RequiredArgsConstructor
public class UserRepositoryImpl {

	private final JPAQueryFactory q;

	public List<User> findRecruitmentNotifiers(final RecruitmentNotifierCondition condition) {
		return q.select(user)
				.from(user)
				.where(
						user.id.in(
								select(notificationKeywordCategory.user.id)
										.from(notificationKeywordCategory)
										.where(notificationKeywordCategory.category.eq(condition.category()))))
				.where(
						user.id.in(
								select(notificationKeywordPosition.user.id)
										.from(notificationKeywordPosition)
										.where(positionsOrCondition(condition.positions()))))
				.where(
						user.id.in(
								select(notificationKeywordStack.user.id)
										.from(notificationKeywordStack)
										.where(stacksOrCondition(condition.stacks()))))
				.fetch();

	}
}
```

이처럼 각 알림 유형에 대한 알림 대상자가 모두 다르며, 적절한 알림 대상자를 선정하는 것은 알림 기능의 핵심 비즈니스 요구사항이다. 또한 조회로직 또한 복잡하다. 즉, 알림 유형에 부합하는 알림 대상자 조회가 올바르게 수행되는지 테스트할 필요가 있다.

정리하면 조회 로직은 경우에 따라 `Stubbing` 처리할 수 있지만, **조회 로직 자체가 핵심 비즈니스 요구사항에 포함되거나, 조회성 쿼리가 복잡하다면 어떠한 방식으로든 조회 로직을 테스트** 해야한다.

<br><br>

## 클래스 분리
`Testable Code` 를 위해 클래스 분리를 고려해볼 수 있다. 이 때, **클래스를 어떤 관심사 기준으로 분리**할 것인지 고민해봐야 한다.

보다 엄중한 관점으로 `SRP(단일 책임 원칙)`을 적용했을 땐, `Domain + UserStory + Service`로 서비스 클래스를 분리할 수 있을 것이다. 예를들면, `NotificationRecruitmentNotifierFindService`, `NotificationReviewPeerFinishNotifierFindService` 이런식이다. 해당 서비스가 협력으로 수행하고 있는 유저스토리의 요구사항이 변경되었을 때만 변경이 이루어지므로 `SRP`에 부합하다. 하지만 이 방식은 수 많은 유저스토리에 따른 수 많은 클래스가 존재하게 된다. 협업을 하며 개발을 진행하다보니 내가 이전에 맡지 않았던 기능들에 대한 추적 및 유지보수에 어려움을 느꼈다.

<br>

### CQRS 부분적 차용

그러던 중 `CQRS(Command Query Responsibility Segregation)`에서 아이디어를 얻었다. 클래스 분리의 관심사 기준을 `Command(CUD)`, `Query(R)`로 나누는 것이다. 예를들면, `DomainCommandService`, `DomainQueryService` 이런식이다. 그리고 기존의 `DomainService`는 두 클래스를 의존한다.

<img width=500 src="https://github.com/june-777/june-777.github.io/assets/68291395/445d51d6-6eab-4461-9f77-0025521e59b0">

<br>

```java
@Service
@RequiredArgsConstructor
public class NotificationService {

	private final NotificationQueryService notificationQueryService;
	private final NotificationCommandService notificationCommandService;

	// for notification server sent events
	private final SseEmitters sseEmitters;

	public void recruitmentNotice(final Recruitment recruitment) {
		// 알림 대상자 조회
		final List<User> recruitmentNotifiers = notificationQueryService.findRecruitmentNotifier(recruitment);

		// 알림 저장
		final List<RecruitmentNotification> recruitmentNotifications = notificationCommandService.saveRecruitmentNotifications(
				recruitment, RECRUITMENT, recruitmentNotifiers);

		// 실시간 알림 전송 요청
		recruitmentNotifications.forEach(recruitmentNotification -> {
			final User notifier = recruitmentNotification.getNotifier();
			final NotificationResponse notificationResponse = new NotificationResponse(recruitmentNotification);
			sseEmitters.sendNotification(notifier, notificationResponse);
		});
	}
    // something public method ...
}
```

```java
@Service
@RequiredArgsConstructor
public class NotificationQueryService {

	private final UserRepositoryImpl userRepository;

	public List<User> findRecruitmentNotifier(final Recruitment recruitment) {
		final RecruitmentNotifierCondition recruitmentNotifierCondition = new RecruitmentNotifierCondition(
				recruitment.getCategory(),
				recruitment.getPositions(),
				recruitment.getStacks());
		log.info("recruitmentNotifierCondition = {}", recruitmentNotifierCondition);
		return userRepository.findRecruitmentNotifiers(recruitmentNotifierCondition);
	}
}
```

```java
@Service
@RequiredArgsConstructor
public class NotificationCommandService {

	private final RecruitmentNotificationRepositoryImpl recruitmentNotificationRepository;

	@Transactional
	public List<RecruitmentNotification> saveRecruitmentNotifications(final Recruitment actor,
																	  final NotificationEventType notificationType,
																	  final List<User> notifiers
	) {
		final List<RecruitmentNotification> recruitmentNotifications = notifiers
				.stream()
				.map(notifier -> RecruitmentNotification.of(notificationType, LocalDateTime.now(), actor, notifier))
				.toList();

		return recruitmentNotificationRepository.saveAll(recruitmentNotifications);
	}
}
```

<br>

**컴포넌트 아키텍처에 대한 전반적 설계**

<img width=600 src="https://github.com/june-777/june-777.github.io/assets/68291395/b12e7f05-bd0d-45fa-a3ea-4eff0c52ee99">

<br><br>

### 클래스 분리를 통해 얻은 이점
**명확한 관심사 분리를 통한 테스트 용이성**  
`CQRS`를 프로젝트에 제대로 적용했다고 보긴 어렵다. 하지만 `CQRS`의 아이디어를 부분적으로 차용함으로서 테스트 용이함의 이점을 얻었다고 생각한다.

조회 로직에 대한 테스트가 필요하면, `DomainQueryService`만 테스트하면 된다.   
또한 `DomainService`에서 핵심 비즈니스 로직에 대해 테스트가 필요하면, 조회 로직은 `Stubbing`처리하면 된다. 조회 로직에 대한 검증은 필요시에 테스트가 이루어졌을 것이기 때문이다.

<br><br>

### 서비스가 서비스를 의존하는 구조에 대해
리팩터링 전, 백엔드 팀원간 회의에서 `서비스가 서비스를 의존하는 구조는 안티패턴 아닌가?` 에 대해 논의가 진행되었다. 이 부분은 조금 더 경험치가 축적돼야 명확해질 것 같지만, 현재로서 내린 결론은 다음과 같다.

<img width=600 src="https://github.com/june-777/june-777.github.io/assets/68291395/b57ad7c5-7673-4bf9-ac4c-109c250e5118">

서비스 계층을 depth로 나누어 생각해볼 수 있는데, **같은 depth의 서비스간 참조가 아니라면 괜찮다**이다.
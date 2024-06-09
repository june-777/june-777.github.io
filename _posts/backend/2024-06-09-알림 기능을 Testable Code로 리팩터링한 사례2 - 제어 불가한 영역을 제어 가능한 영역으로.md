---
layout: single
title:  "알림 기능을 Testable Code로 리팩터링한 사례2 - 제어 불가한 영역을 제어 가능한 영역으로"
toc: true
toc_sticky: true
---
> 알림 기능을 구현하며 테스트 용이한 코드로 리팩터링한 사례를 공유하고자 합니다.  
> 다소 정답이 없는 설계적 영역이므로, 주관적 견해인 점을 참고하시면 좋을 것 같습니다.  
> 편의를 위해 비격식체를 사용합니다.

<br>

## 테스트 어려운 코드
내부 구현이 제어할 수 없는 영역을 의존하고 있는 경우, 테스트하기 어려운 코드가 된다. 아래의 예시 코드를 살펴보자.

`NotificationQueryService` 객체는 알림과 관련된 조회를 책임지고 있는 객체이다.  
그 중 `findStudyEndDateNotifier()` 메서드는 `스터디 종료 기간 알림 유형`에 맞는 `알림 대상자`를 조회하는 기능이다.  
알림 대상자는 `스터디 종료 시간`이 현재 시각 기준 5일 남은 스터디들의 스터디장이다.

```java
@Service
@RequiredArgsConstructor
public class NotificationQueryService {
	
	private final ParticipantRepositoryImpl participantRepository;
	// 생략

	public List<Participant> findStudyEndDateNotifier() {
		final Long remainingPeriod = 5L;
		final LocalDate endDate = LocalDateTime.now().toLocalDate() // 여기가 문제
				.plusDays(remainingPeriod);
		final LocalDateTime startOfDay = endDate.atStartOfDay();
		final LocalDateTime endOfDay = endDate.atTime(LocalTime.MAX);

		return participantRepository.findOwnerParticipantsBetweenDateRange(
				new StudyEndDateNotifierCond(Role.OWNER, startOfDay, endOfDay));
	}

	// 생략
}
```

이 코드는 테스트하기 어려운 코드이다. 날짜와 관련된 `LocalDateTime`을 **직접 의존**함으로 인해, **테스트 코드상에서 날짜를 제어할 수 없기 때문**이다.

<br><br>

## 인터페이스 분리
제어할 수 없는 영역을 **인터페이스로 분리**하여 제어할 수 있는 영역으로 만들 수 있다.

```java
public interface UtcDateTimePicker {
    LocalDateTime now();
}
```

```java
@Profile("!test")
@Component
public class CurrentUtcDateTimePicker implements UtcDateTimePicker {

	public LocalDateTime now() {
		return LocalDateTime.now(ZoneOffset.UTC).truncatedTo(ChronoUnit.MICROS);
	}
}
```

```java
@Profile("test")
@Component
public class FixedUtcDateTimePicker implements UtcDateTimePicker {

	public static final LocalDateTime DEFAULT_FIXED_UTC_DATE_TIME = defaultFixedUtcDateTime();
	private LocalDateTime fixedUtcDateTime = DEFAULT_FIXED_UTC_DATE_TIME;

	@Override
	public LocalDateTime now() {
		return fixedUtcDateTime;
	}

	private static LocalDateTime defaultFixedUtcDateTime() {
		return LocalDateTime.of(2000, 1, 1, 0, 0, 0, 0);
	}
}
```

<br><br>

풀어서 말하면, 구체적인 구현체를 의존(강결합)함으로서 제어할 수 없게 된 영역을 **인터페이스를 의존(약결합)하게 함으로서 제어할 수 있는 영역으로** 만들어주는 것이다. 테스트 상에는 인터페이스를 구현한 **`Fake 객체`** 를 사용함으로서 제어할 수 있게 된다. 위의 `FixedUtcDateTimePicker`가 Fake 객체에 해당한다.

```java
@Service
@RequiredArgsConstructor
public class NotificationQueryService {

	private final ParticipantRepositoryImpl participantRepository;
	private final UtcDateTimePicker utcDateTimePicker; // 인터페이스 의존(약결합)
	// 생략

	public List<Participant> findStudyEndDateNotifier() {
		final Long remainingPeriod = 5L;
		final LocalDate endDate = utcDateTimePicker.now().toLocalDate()
				.plusDays(remainingPeriod);
		final LocalDateTime startOfDay = endDate.atStartOfDay();
		final LocalDateTime endOfDay = endDate.atTime(LocalTime.MAX);

		List<Participant> ownerParticipantsBetweenDateRange = participantRepository.findOwnerParticipantsBetweenDateRange(
				new StudyEndDateNotifierCond(Role.OWNER, startOfDay, endOfDay));
		log.info("ownerParticipantsBetweenDateRange = {}", ownerParticipantsBetweenDateRange);
		return ownerParticipantsBetweenDateRange;
	}

	// 생략
}
```

<br> <br>
이를 그림으로 표현하면 다음과 같다.

**before**

<img width=600, src="https://github.com/june-777/june-777.github.io/assets/68291395/25d40a97-582e-4676-bed7-b4bbd3091b37">


<br><br>

**after**

<img width=600, src="https://github.com/june-777/june-777.github.io/assets/68291395/ef267bf4-47ee-4b58-81d8-24cdb712ffd6">


<br><br>

## 테스트 코드
`FixedDateTimePicker`의 `now()`는 `2000년 1월 1일 0시 0분 0초 0나노초 고정된 시간`을 반환한다.  
따라서 아래와 같은 테스트 코드 작성이 가능해졌다.

```java
// Pseudo Code

@Test
@DisplayName("[Success] 현재 시각이 2000:01:06:00:00:00일 때, "
		+ "종료기간이 2000:01:11:00:00:00 ~ 2000:11:59:59:999999인 스터디들의 스터디장들을 알림 대상자로 조회")
@Transactional
void studyEndDateInRangeTest() {
	LocalDateTime fixedDateTime = FixedUtcDateTimePicker.DEFAULT_FIXED_UTC_DATE_TIME;
	Study study1 = StudyFixture.STUDY1(
			user1, fixedDateTime,
			LocalDateTime.of(fixedDateTime.getYear(), fixedDateTime.getMonth(),
					fixedDateTime.getDayOfMonth() + 5, 10, 0, 0, 0)
			);
	Study study2 = StudyFixture.STUDY2(
			user2,
			fixedDateTime,
			LocalDateTime.of(fixedDateTime.getYear(), fixedDateTime.getMonth(),
					fixedDateTime.getDayOfMonth() + 5, 23, 59, 59, 999999)
			);

	Participant ownerOfStudy1 = Participant.from(study1, user1, backend, OWNER);
	Participant memberOfStudy1 = Participant.from(study1, user1, backend, OWNER);
	Participant ownerOfStudy2 = Participant.from(study2, user2, backend, OWNER);

	List<Participant> studyEndDateNotifier = notificationQueryService.findStudyEndDateNotifier();

	assertThat(studyEndDateNotifier).contains(ownerOfStudy1, ownerOfStudy2);
}
```
---
title: "TalleMalle"
date: 2026-03-15
draft: false
summary: "같은 방향으로 이동하는 사용자끼리 택시비를 분담하도록 매칭해주는 실시간 위치 기반 동승 매칭 커뮤니티 서비스입니다. 출발지·도착지를 검색해 모집글을 올리면 지도에서 주변 모집글을 실시간으로 확인하고 참여할 수 있으며, 정원이 차면 자동 마감되는 상태 머신으로 모집부터 기사 호출까지의 흐름을 관리합니다. 백엔드는 지도 바운더리 기반 조회 최적화, 동시성 제어, 부하 테스트 기반 성능 개선에 집중했습니다."
role: "팀원 5명 · 백엔드 담당"
period: "2026.01 ~ 2026.04 (BEYOND SW 캠프 24기 3차 프로젝트)"
accent: "blue"
demo: "http://www.tallemalletest.kro.kr"
repo: "https://github.com/DongHyunj/TalleMalle"
techStack: ["Java 17", "Spring Boot 3.5.10", "Spring Data JPA", "Vue 3", "Nginx", "WebSocket (STOMP)", "MariaDB", "AWS EC2", "Ubuntu", "nGrinder"]
contributions:
  - category: "Backend"
    items:
      - "모집글 CRUD API 설계 및 구현"
      - "위치 기반 검색"
      - "모집글 동시성 제어"
      - "모집글 상태 머신 설계"
      - "스케줄러 기반 자동화"
      - "Spring Event 기반 실시간 브로드캐스트"
      - "비즈니스 규칙 검증"
  - category: "Frontend"
    items:
      - "지도 기반 메인 UI 구현 (Vue 3 + Kakao Maps API)"
      - "실시간 모집글 동기화 (STOMP WebSocket)"
      - "모집글 생성 모달"
      - "슬라이딩 패널 UI"
      - "무한 스크롤 & 클라이언트 필터링"
---

## 메인 화면 (담당 영역)

저는 서비스의 핵심 진입점인 **메인 화면**을 담당했습니다. 카카오 지도 API를 활용해 사용자 위치를 기반으로 주변 모집글을 지도 위에 실시간으로 표시했습니다.

- **좌측 패널** - 모집글 검색 및 목록 조회
- **지도** - 위치별 마커를 클릭해 모집글 상세 확인
- **우측 하단** - 원하는 시간대로 모집글 직접 생성

![TalleMalle 메인 화면 - 지도와 실시간 모집글](images/projects/tallemalle/main.png)

### 트러블슈팅 1. 모집글 목록 조회 N+1 - 쿼리 1001회를 2회로

#### 문제 상황

초기에는 메인 화면 진입 시 **모집 중인 모집글 전체를 조회**하도록 구현했습니다. 하지만 데이터가 쌓일수록 목록이 뜨는 속도가 눈에 띄게 느려졌고, 실제 부하 상황을 확인하기 위해 nGrinder로 부하 테스트를 진행했습니다.

- **테스트 조건**: Vuser 52명 (Agent 2 × Process 2 × Thread 13), 3분간 1초 주기로 모집글 전체 조회, **모집글 1000건** 기준
- **결과**: 평균 처리량 **TPS 2.3**, 평균 응답 시간 **21.3초**

초당 2.3건밖에 처리하지 못했고, 사용자 입장에서는 **리스트 탭을 누르고 21초 동안 화면이 멈춰 있는** 치명적인 상태였습니다.

![nGrinder TPS (개선 전) - 평균 약 2.3](images/projects/tallemalle/ngrinder-tps.png)

#### 원인 분석

백엔드 쿼리 로그를 확인했습니다. 모집(Recruit) 엔티티는 방장을 `@ManyToOne`(다대일), 참여자 목록을 `@OneToMany`(일대다)로 매핑하고 있었습니다. 응답 DTO로 변환할 때 **모집글마다 참여자 컬렉션(LAZY)에 접근**하면서, 모집글 1건당 참여자 조회 쿼리가 따로 나가는 **N+1 문제**가 발생하고 있었습니다. 모집글 1000건이면 `목록 1회 + 참여자 1000회 = 1001회`의 쿼리가 실행된 것입니다.

흥미로운 점은 **방장(owner)은 N+1을 일으키지 않았다**는 것입니다. DTO에서 `owner.getIdx()`로 식별자만 사용했는데, LAZY 프록시는 식별자 조회에 추가 쿼리를 보내지 않기 때문입니다(FK인 `owner_id`가 이미 `recruits` row에 있음). 실제 로그에도 `participations` 조회만 반복될 뿐 `users` 조회는 없었고, 이로써 **원인을 참여자 컬렉션으로 특정**할 수 있었습니다.

![N+1 로그 - participations 조회 반복](images/projects/tallemalle/nplus1-log.png)

#### 해결 과정

가장 먼저 떠올린 해법은 `JOIN FETCH`였습니다. 연관 데이터를 한 쿼리에 끌어오면 N+1이 사라질 테니까요. 그래서 참여자 컬렉션까지 fetch join을 걸어봤지만, 로그에서 새로운 문제를 발견했습니다.

![카테시안 곱 로그 - select distinct + left join participations](images/projects/tallemalle/cartesian-log.png)

`left join participations` 때문에 **모집글 한 건이 참여자 수만큼 row로 복제**됐고(카테시안 곱), 중복을 지우기 위해 `select distinct`가 붙어 있었습니다. 더 큰 문제는 제가 사용하던 `Slice` 페이징이 **DB가 아닌 메모리에서 처리**된다는 점이었습니다(`firstResult/maxResults specified with collection fetch; applying in memory`). 즉, **1:N 컬렉션은 fetch join으로 풀면 안 된다**는 결론에 도달했습니다.

그래서 **연관 관계의 성격에 따라 전략을 분리**했습니다.

- **방장** (다대일, to-one) → `JOIN FETCH`

  row가 증식하지 않는 to-one 연관이므로 단일 쿼리에 안전하게 함께 로드
- **참여자** (일대다, 컬렉션) → `@BatchSize(size = 100)`

  fetch join 대신 `WHERE recruit_idx IN (?, ?, ...)` 형태로 **한 번에 배치 조회**, 페이징도 그대로 유지
- **목록 DTO 경량화** → 참여자의 `id`만 노출

  user·profile 엔티티를 아예 로딩하지 않아 **2차 N+1까지 차단** (이름·프로필이 필요한 채팅방 멤버 조회는 별도 `JOIN FETCH` 쿼리로 분리)

#### 결과

| 구분 | 쿼리 수 |
|------|---------|
| 개선 전 (`findAll` + LAZY) | **1001회** (1 + N) |
| Fetch Join (컬렉션 포함) | 1회이나 카테시안 곱 + 메모리 페이징 |
| **최종** (JOIN FETCH + @BatchSize) | **2회로 고정** |

- 모집글 수와 무관하게 쿼리 **2회로 고정**
- nGrinder 동일 조건 재측정: **TPS 2.3 → 약 9** (약 4배 ↑), **평균 응답 시간 21.3초 → 약 5초** (약 4배 ↓)

![nGrinder TPS (개선 후) - 약 9](images/projects/tallemalle/ngrinder-tps-after.png)

### 트러블슈팅 2. 지도 전체 조회 → 화면 경계 조회 + 디바운싱

#### 문제 상황

초기 구현에서는 메인 화면에서 모집 중인 모집글을 전체 조회한 뒤, 프론트엔드에서 지도 영역에 맞게 필터링하는 방식이었습니다. 기능은 동작했지만 두 가지 비효율이 있었습니다.

- **화면 밖 데이터까지 전송** - 사용자가 보고 있는 지도 영역과 무관하게 전국의 모집글을 모두 내려받아, 데이터가 쌓일수록 응답 크기가 선형으로 커졌습니다.
- **지도 이동마다 과잉 요청** - 지도는 이동·확대·축소가 잦은데, 이동이 멈출 때마다 발생하는 `idle` 이벤트가 연속으로 수십 번 발생해 짧은 시간에 API가 반복 호출됐습니다. 서버 부하는 물론 카카오 지도 API 쿼터까지 위협하는 구조였습니다.

#### 해결 과정

화면에 보이는 만큼만, 이동이 끝났을 때만 조회한다는 목표로 세 가지를 적용했습니다.

**① 지도 경계 좌표 기반 조회**

프론트엔드에서 지도의 남서(SW)·북동(NE) 좌표를 파라미터로 전송하고, 백엔드는 위·경도 범위 조건으로 화면 안에 있는 모집글만 조회하도록 변경했습니다.

```java
// RecruitRepository
@Query("SELECT r FROM Recruit r JOIN FETCH r.owner " +
       "WHERE r.startLat BETWEEN :swLat AND :neLat " +
       "AND r.startLng BETWEEN :swLng AND :neLng " +
       "AND r.status != :status")
Slice<Recruit> findActiveRecruitsInBounds(...);
```

```js
// Map.vue — 지도 이동이 멈추면 현재 경계 좌표를 부모로 전달
window.kakao.maps.event.addListener(map, 'idle', () => {
  const bounds = map.getBounds()
  emit('bounds-changed', {
    swLat: bounds.getSouthWest().getLat(), swLng: bounds.getSouthWest().getLng(),
    neLat: bounds.getNorthEast().getLat(), neLng: bounds.getNorthEast().getLng(),
  })
})
```

**② 디바운싱** (300ms)

지도 이동 중 연속으로 발생하는 요청을 막기 위해, 마지막 이벤트로부터 0.3초간 추가 입력이 없을 때 단 1회만 API를 호출하도록 처리했습니다.

```js
// Main.vue
let mapSearchTimeout = null
const handleSearchRecruits = (bounds) => {
  if (mapSearchTimeout) clearTimeout(mapSearchTimeout) // 이전 예약 취소
  mapSearchTimeout = setTimeout(async () => {
    const res = await api.searchRecruits({ ...bounds, page, size })
    // ...
  }, 300)
}
```

**③ Slice 페이징 + 무한 스크롤**

경계 안의 결과도 한 번에 다 내리지 않고, `Page` 대신 `Slice`를 사용해 COUNT 쿼리 없이 `limit + 1`로 다음 페이지 존재 여부만 판단하도록 했습니다. 사용자가 목록을 스크롤할 때 다음 페이지를 이어서 로드합니다.

#### 결과

- **화면 밖 데이터 전송 제거** → 응답 페이로드와 조회 대상이 지도 영역 단위로 한정되어, 전체 모집글 수가 늘어도 조회 비용이 일정하게 유지
- **지도 이동 중 다수 요청을 최종 1회로 축소** → 불필요한 네트워크 요청 약 90% 차단
- **카카오 지도 API 호출량 감소** → 쿼터 여유 확보

같은 전체 조회 설계에서 출발한 문제였지만, 트러블슈팅 1이 **레코드당 쿼리 수**(N+1)를 줄인 것이라면, 트러블슈팅 2는 **조회하는 레코드 수와 요청 빈도 자체**를 줄인 개선이었습니다. 데이터가 많아질수록 둘의 효과가 함께 커지는 구조를 만든 것이 핵심입니다.

![지도 조회 흐름 - 디바운싱 → Bounds 공간 쿼리 → Slice 응답](images/projects/tallemalle/bounds-sequence.png)

`Vue 3` `Kakao Maps API` `Debouncing` `Spring Data JPA` `Slice` `위경도 범위 쿼리`

### 트러블슈팅 3. 동시 참여 요청의 동시성 제어 - 오버부킹 차단

#### 문제 상황

택시 합승은 **정원이 정해진 한정 자원**입니다. 정원 4명 중 마지막 1자리가 남은 모집글에 **여러 명이 동시에 참여 버튼을 누르는** 상황이 충분히 발생할 수 있었습니다. 단순한 조회 후 비교 로직은 이 경우 위험합니다. 두 요청이 거의 동시에 들어오면 **둘 다 아직 자리 있음으로 통과**하는 **경쟁 상태**(race condition)가 생깁니다.

이를 검증하기 위해 한 모집글에 동시 참여 요청을 몰아 부하 테스트를 진행하자, 두 가지 사고가 관측됐습니다.

**사고 1. 데드락으로 인한 500 에러**

여러 트랜잭션이 동시에 락을 확보하려다 서로의 락을 기다리는 교착 상태(deadlock)에 빠졌고, 일부 요청이 강제 롤백되며 Vuser에게 500 에러가 반환됐습니다.

```text
java.sql.SQLTransactionRollbackException: (conn=84) Deadlock found when trying to get lock; try restarting transaction
	at org.mariadb.jdbc.export.ExceptionFactory...
```

**사고 2. 갱신 손실로 인한 인원수 불일치**

`participations` 테이블에는 **4명**이 들어갔는데 `recruits.current_capacity` 카운터는 **2**에 그쳐 데이터 정합성이 깨졌습니다. 동시에 들어온 트랜잭션들이 **같은 값을 읽고 각자 +1 해 덮어쓰면서**(lost update) 증가분이 유실된 것입니다.

![문제 - participations 4건 (실제 참여자)](images/projects/tallemalle/join-before-participations.png)

![문제 - current_capacity 2 (실제 4와 불일치)](images/projects/tallemalle/join-before-capacity.png)

#### 락 전략 의사결정

동시성 제어 방식을 두고 낙관적 락과 비관적 락을 비교했습니다.

| 방식 | 동작 | 이 상황에서의 판단 |
|------|------|---------------------|
| 낙관적 락 (`@Version`) | 충돌을 사후 감지 → 예외 발생 → 재시도 | 마지막 한 자리에 요청이 몰리면 **충돌이 잦아** 재시도 비용·실패율이 큼 ❌ |
| **비관적 락** (`PESSIMISTIC_WRITE`) | DB 행 배타락으로 동시 요청을 **직렬화** | 임계 영역이 짧고 충돌이 잦은 합승 시나리오에 적합 ✅ **채택** |

티켓팅처럼 **충돌 빈도가 높은 환경**에서는 충돌을 사후에 되돌리는 낙관적 락보다, 처음부터 한 줄로 세우는 비관적 락이 더 안정적이라고 판단했습니다.

#### 해결 과정

참여 처리 시 모집글 행에 **배타락**(`SELECT ... FOR UPDATE`)을 걸어 동시 요청을 직렬화했습니다.

```java
// RecruitRepository
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT r FROM Recruit r WHERE r.idx = :idx")
Optional<Recruit> findByIdForUpdate(@Param("idx") Long idx);
```

```java
// RecruitService.join() — 락 획득 후 검사·증가가 하나의 임계 영역에서 수행됨
@Transactional
public boolean join(AuthUserDetails user, Long recruitIdx) {
    Recruit recruit = recruitRepository.findByIdForUpdate(recruitIdx)  // 🔒 배타락 획득
            .orElseThrow(...);
    // 정원 검사
    if (recruit.getStatus() == RecruitStatus.FULL
            || recruit.getCurrentCapacity() >= recruit.getMaxCapacity()) {
        throw new BaseException(BaseResponseStatus.RECRUIT_FULL);
    }
    // ... 중복 참여 확인 ...
    recruit.addParticipant();  // 인원 +1, 정원 도달 시 FULL 전환
    return true;
}
```

핵심은 **정원 검사 → 인원 증가가 락 안에서 하나의 단위로 처리**된다는 점입니다. 먼저 락을 획득한 요청이 커밋하기 전까지 다른 요청은 대기하므로, 뒤 요청은 **갱신된 최신 인원**을 보고 정확히 차단됩니다. 또한 모든 요청이 **모집글 행 하나에 대한 락을 동일한 지점에서 가장 먼저 획득**하므로, 락 획득 순서가 어긋나며 생기던 데드락의 여지도 사라졌습니다.

#### 결과

비관적 락 적용 후에는 동시 요청이 몰려도 **데드락과 갱신 손실이 모두 사라졌고**, `current_capacity`가 실제 참여자 수와 **정확히 일치**(4 = 4)하여 정원 초과 없이 정확히 마감되고 모집 인원 데이터의 **정합성이 보장**됩니다.

![개선 후 - participations 4건 (실제 참여자)](images/projects/tallemalle/join-after-participations.png)

![개선 후 - current_capacity 4 (정합성 일치)](images/projects/tallemalle/join-after-capacity.png)

![동시 참여 락 흐름 - A 락 획득·커밋 → B·C 대기 후 갱신값 읽고 차단](images/projects/tallemalle/concurrency-sequence.png)

`Spring Data JPA` `Pessimistic Lock` `@Transactional` `Race Condition` `동시성 제어`

### 트러블슈팅 4. 모집글 상태 변경 실시간 동기화 - WebSocket + STOMP 브로드캐스트

#### 문제 상황

지도 위 모집글은 **여러 사용자가 동시에 보고 있는 공유 화면**입니다. A가 참여해 정원이 차거나 모집글이 마감돼도 B·C의 지도에는 여전히 참여 가능으로 보이면, 이미 사라진 자리에 참여를 시도하다 실패하는 혼선이 생깁니다. 모집글의 **상태 변경(참여·생성·마감)을 구독 중인 모든 클라이언트에게 즉시 전파**할 방법이 필요했습니다.

가장 단순한 방법은 클라이언트가 주기적으로 다시 조회하는 폴링이지만, 통신 방식을 비교한 끝에 폴링을 배제했습니다.

| 방식 | 특징 | 판단 |
|------|------|------|
| Short Polling | 주기적 HTTP 요청 | 잦은 요청으로 서버 부하가 크고 실시간성 보장이 어려움 ❌ |
| SSE (Server-Sent Events) | 서버→클라이언트 단방향 | 단방향이라 향후 채팅 등 양방향 기능 확장에 부적합 ❌ |
| **WebSocket + STOMP** | 양방향 + Pub/Sub 라우팅 | 구독 기반 메시지 라우팅으로 실시간 전파에 적합 ✅ **채택** |

순수 WebSocket은 메시지 포맷과 라우팅을 직접 구현해야 하는 부담이 큽니다. 반면 **STOMP**는 Pub/Sub(발행·구독) 모델을 기본 지원해 `/topic/all-calls`(전체 지도)나 `/topic/chat/{id}`(특정 방) 단위의 효율적인 메시지 라우팅이 가능했습니다.

#### 해결 과정

모집글 상태가 바뀌면 그 변경을 **구독 중인 모든 클라이언트에게 STOMP로 브로드캐스트**하도록 구성했습니다.

- **발행 (Publisher)** - 모집글 상태 변경(참여·생성·마감) 이벤트가 발생하면 서버가 메시지를 발행
- **메시지 브로커 (STOMP)** - `/topic/all-calls` 채널을 구독 중인 클라이언트에게 페이로드를 라우팅
- **구독 (Subscriber)** - 지도를 보고 있는 모든 클라이언트가 이 채널을 구독하고, 수신 즉시 마커·목록을 갱신

특히 DB 트랜잭션과 메시지 발행의 정합성을 위해, 참여·마감 같은 변경이 **DB에 실제로 커밋된 뒤에만**(`AFTER_COMMIT`) 브로드캐스트가 나가도록 Spring Event로 분리했습니다. 트랜잭션이 롤백되면 잘못된 상태가 전파되는 일이 없습니다.

#### 결과

- 한 사용자의 참여·마감이 **같은 화면을 보는 모든 사용자에게 즉시 반영**되어, 이미 마감된 모집글에 참여를 시도하는 혼선을 제거
- `/topic/all-calls`(지도 전체)와 `/topic/chat/{id}`(채팅방) 채널을 분리해, 같은 STOMP 인프라 위에서 **실시간 지도 동기화와 채팅을 함께 확장**할 수 있는 구조 확보

![실시간 브로드캐스트 흐름 - 상태 변경 이벤트 → STOMP /topic/all-calls → 구독자 일괄 수신](images/projects/tallemalle/socket-sequence.png)

`WebSocket` `STOMP` `Pub/Sub` `Spring Event` `실시간 동기화`


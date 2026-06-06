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

## 🗺️ 메인 화면 (담당 영역)

저는 서비스의 핵심 진입점인 **메인 화면**을 담당했습니다. 카카오 지도 API를 활용해 사용자 위치를 기반으로 주변 모집글을 지도 위에 실시간으로 표시했습니다.

- **좌측 패널** - 모집글 검색 및 목록 조회
- **지도** - 위치별 마커를 클릭해 모집글 상세 확인
- **우측 하단** - 원하는 시간대로 모집글 직접 생성

![메인 화면 - 지도 기반 주변 모집글 조회](images/projects/tallemalle/main.png)

### 🔧 트러블슈팅 1. 모집글 목록 조회 N+1 - 쿼리 1001회를 2회로

#### 문제 상황

초기에는 메인 화면 진입 시 **모집 중인 모집글 전체를 조회**하도록 구현했습니다. 하지만 데이터가 쌓일수록 목록이 뜨는 속도가 눈에 띄게 느려졌고, 실제 부하 상황을 확인하기 위해 nGrinder로 부하 테스트를 진행했습니다.

- **테스트 조건**: Vuser 52명 (Agent 2 × Process 2 × Thread 13), 3분간 1초 주기로 모집글 전체 조회, **모집글 1000건** 기준
- **결과**: 평균 처리량 **TPS 2.3**, 평균 응답 시간 **21.3초**

초당 2.3건밖에 처리하지 못했고, 사용자 입장에서는 **리스트 탭을 누르고 21초 동안 화면이 멈춰 있는** 치명적인 상태였습니다.

![nGrinder TPS (개선 전) - 평균 약 2.3](images/projects/tallemalle/ngrinder-tps.png)

![nGrinder 평균 응답 시간 / Vuser (개선 전) - 약 21초 · Vuser 52](images/projects/tallemalle/ngrinder-time.png)

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

![nGrinder 평균 응답 시간 / Vuser (개선 후) - 약 5초](images/projects/tallemalle/ngrinder-time-after.png)


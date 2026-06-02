---
title: "TalleMalle"
date: 2026-03-15
draft: false
summary: "같은 방향으로 이동하는 사용자끼리 택시비를 분담하도록 매칭해주는 실시간 위치 기반 동승 매칭 커뮤니티 서비스"
role: "팀 프로젝트 · 백엔드"
period: "한화 BEYOND SW 캠프 4차 프로젝트"
accent: "blue"
tech: ["Spring Boot", "Spring Data JPA", "WebSocket+STOMP", "Vue 3", "MariaDB", "nGrinder"]
demo: "http://www.tallemalletest.kro.kr"
repo: "https://github.com/beyond-sw-camp/be24-4th-saraITne-TalleMalle"
---

## 개요

같은 방향으로 이동하는 사용자끼리 택시비를 분담하도록 **실시간으로 매칭**해주는 위치 기반 동승 매칭 커뮤니티 서비스입니다. 출발지·도착지가 비슷한 사용자를 묶어 동승 모집글을 만들고, 실시간 채팅으로 빠르게 약속을 잡을 수 있습니다.

## 주요 기능

- **위치 기반 동승 매칭** — 출발지/도착지 좌표를 기준으로 같은 방향 사용자를 매칭
- **실시간 채팅** — WebSocket + STOMP 기반 그룹 채팅으로 즉시 소통
- **모집글 관리** — 동승 모집글 작성·조회·신청·마감 등 전체 흐름 구현
- **커뮤니티** — 후기·평점으로 신뢰 기반 매칭 지원

## 맡은 역할 & 기여

- **모집글 조회 TPS 3.8배 개선** — 조회 쿼리·인덱스 최적화로 메인 트래픽 구간 성능 개선
- **Pessimistic Lock 동시성 제어** — 동시에 같은 모집글에 신청이 몰릴 때 정원 초과·중복 매칭을 방지
- **nGrinder 부하 테스트** — 시나리오 기반 부하 테스트로 병목 구간을 식별하고 개선 효과를 정량 검증

## 기술 스택

- **Backend** — Spring Boot, Spring Data JPA, WebSocket + STOMP
- **Frontend** — Vue 3
- **Database / Test** — MariaDB, nGrinder

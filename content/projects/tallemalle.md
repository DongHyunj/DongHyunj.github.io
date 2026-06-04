---
title: "TalleMalle"
date: 2026-03-15
draft: false
summary: "같은 방향으로 이동하는 사용자끼리 택시비를 분담하도록 매칭해주는 실시간 위치 기반 동승 매칭 커뮤니티 서비스"
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

## 개요

같은 방향으로 이동하는 사용자끼리 택시비를 분담하도록 **실시간으로 매칭**해주는 위치 기반 동승 매칭 커뮤니티 서비스입니다. 출발지·도착지가 비슷한 사용자를 묶어 동승 모집글을 만들고, 실시간 채팅으로 빠르게 약속을 잡을 수 있습니다.

지도 바운더리 기반으로 주변 모집글을 실시간으로 조회하고, 정원이 차면 자동 마감되는 모집글 상태머신으로 라이프사이클을 관리합니다. 동시성 제어와 성능 최적화에 집중한 백엔드 설계가 핵심입니다.

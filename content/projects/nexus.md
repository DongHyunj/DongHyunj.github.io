---
title: "Nexus (SCM)"
date: 2026-05-20
draft: false
summary: "프랜차이즈 본사가 가맹점의 매출·재고·발주를 통합 관리하는 B2B 공급망 주문관리 시스템 (SaaS)"
role: "팀 프로젝트 · 백엔드 / 인프라"
period: "한화 BEYOND SW 캠프 최종 프로젝트"
accent: "pink"
tech: ["Spring Boot", "SSE", "Kafka", "Redis Cluster", "Kubernetes", "k6"]
demo: "https://www.nexusscm.kro.kr"
repo: "https://github.com/beyond-sw-camp/be24-fin-Fiveguys-Nexus"
---

## 개요

프랜차이즈 본사가 가맹점의 **매출·재고·발주를 통합 관리**하는 B2B 공급망 주문관리(SCM) SaaS입니다. 본사와 다수 가맹점 사이의 주문 흐름을 표준화하고, 실시간 알림과 통계 대시보드로 운영 가시성을 높였습니다.

## 주요 기능

- **발주 / 재고 / 매출 관리** - 가맹점 발주부터 본사 재고 차감·정산까지 일괄 처리
- **실시간 알림** - SSE를 직접 구현해 발주 상태 변경을 즉시 푸시
- **통계 대시보드** - 매출·발주 집계를 통계 MSA로 분리해 비동기 처리

## 맡은 역할 & 기여

- **통계 집계 MSA 5단계 진화 주도** - 모놀리식 동기 처리 → 이벤트 기반 비동기 처리까지 단계적으로 진화시키고, **k6 부하 테스트**로 각 단계의 개선 효과를 검증
- **Kafka 메시지 정합성 확보** - `@TransactionalEventListener(AFTER_COMMIT)`로 커밋 이후에만 발행하도록 바꿔 phantom 메시지 **4건 → 0건**
- **Consumer race condition 해결** - `@SQLInsert ON DUPLICATE KEY UPDATE`로 동시 소비 시 누락 **500건 → 0건**
- **Redis Cluster(3+3) 직접 구축** - Hash Tag 슬롯 설계로 키 분산·원자성 보장
- **Kubernetes Blue/Green 무중단 배포** - StatefulSet·Ingress TLS 포함 5노드 클러스터 직접 운영

## 기술 스택

- **Backend** - Spring Boot, SSE, Kafka
- **Infra** - Redis Cluster, Kubernetes, k6

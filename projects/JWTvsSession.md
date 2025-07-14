---
layout: page
title: JWT vs Session 인증 비교 및 기술 선택 이유
description: >
  OSID 프로젝트에서 Spring Security + JWT를 선택한 배경과 세션 기반 인증 대비 장단점을 정리합니다.
hide_description: true
sitemap: false
---
# JWT vs Session 인증 비교 및 기술 선택 이유

<img src="/assets/img/blog/JWTvsSession.png" alt="JWT vs Session 비교 차트" style="max-width:100%; height:auto; margin: 1em 0;" />

| 비교 항목       | Spring Security + JWT         | Session 기반 인증                       |
| ----------- | ----------------------------- | ----------------------------------- |
| **구조**      | Stateless (서버에 토큰 저장 불필요)     | Stateful (세션 스토리지 필요)               |
| **확장성**     | 우수 (세션 클러스터링 불필요, 인스턴스 추가 용이) | 보통 (세션 공유/클러스터링, Sticky session 필요) |
| **성능**      | 높음 (추가 DB 조회 없이 토큰 검증)        | 보통 (세션 조회를 위한 Redis/DB 호출)          |
| **보안성**     | 우수 (토큰 서명 검증, 만료 관리 가능)       | 양호 (세션 탈취 시 무효화 가능)                 |
| **관리 오버헤드** | 낮음 (토큰 자체 관리, 서버 리소스 부담 없음)   | 높음 (세션 저장소 운영 및 모니터링 필요)            |
| **구현 복잡도**  | 보통 (JWT 발급/검증 로직 추가 필요)       | 낮음 (프레임워크 내장 기능 활용)                 |
| **유연성**     | 높음 (모바일/외부 API 등 다양한 환경 지원)   | 제한적 (CORS/Cookie 정책 제약)             |

---

## 기술 선택 요약

* **Stateless 구조**: 서버 리소스를 절약하고, 다중 인스턴스 환경에서 세션 동기화 없이 확장 가능
* **빠른 인증 처리**: 토큰 자체에 권한 및 만료 정보 포함으로 DB 조회 최소화
* **간편한 운영**: 세션 스토리지 관리 부담 제거, 모바일 및 외부 서비스 연동 시 CORS/Cookie 제약 완화

*위 비교를 통해, OSID 프로젝트에서 JWT 기반 인증이 최적의 솔루션임을 명확히 확인할 수 있습니다.*


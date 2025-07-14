---
layout: page
title: OSID 배포 & CI/CD 최적화
description: >
  AWS Free Tier 제약(EC2 750시간, S3 5GB) 내에서 OSID 시스템의
  CI/CD 파이프라인을 안정적으로 운영하기 위한 문제 정의, 해결 과정,
  성과 및 개선 아이디어를 간략히 정리합니다.
hide_description: true
sitemap: false
---
# OSID 배포 & CI/CD 최적화

<figure style="text-align:center; margin:1em 0;">
  <img src="/assets/img/blog/before-deploy-metrics.png" alt="배포 전 모니터링 지표" style="max-width:80%;"/>
  <figcaption>배포 전 모니터링 지표</figcaption>
</figure>

## 문제 정의
* AWS Free Tier 환경(t2.micro 750시간, S3 5GB) 내에서 CI/CD 운영 필요
* EC2 인스턴스에서 Gradle 빌드 시 CPU 사용률 100% 근접 → 빌드 지연
* Docker 이미지 과다로 빌드·배포 시간·자원 낭비
* 중지되지 않은 컨테이너·스냅샷 과금 위험

## 가설

1. Docker 이미지 경량화 + 병렬 스레드 제한 → CPU 과부하 완화
2. 프론트엔드는 S3 정적 호스팅, 백엔드는 EC2 독립 운영 → 리소스 분리
3. 배포 직후 불필요 리소스 자동 정리 → 과금 위험 제거

## 해결 방안

1. **경량화된 Docker 이미지**: Alpine 기반 멀티스테이지 빌드 적용
2. **병렬 빌드 제한**: `./gradlew -T 1C` 옵션으로 스레드 수 조정
3. **분리된 배포 구조**:

    * 백엔드: EC2 + Docker Compose로 컨테이너 관리
    * 프론트엔드: S3 정적 호스팅 + CloudFront CDN
4. **CI/CD 파이프라인**:

    * 백엔드: GitHub Actions → SSH로 EC2 배포 → `docker-compose up -d`
    * 프론트엔드: GitHub Actions → S3 sync 액션으로 버킷 배포
5. **자동 리소스 정리**:

    * `docker system prune -af`
    * 오래된 EBS 스냅샷 주기적 삭제 스크립트


<!-- 배포 후 모니터링 지표 -->
<figure style="text-align:center; margin:1em 0;">
  <img src="/assets/img/blog/after-optimization-metrics.png" alt="배포 후 모니터링 지표" style="max-width:80%; height:auto;" />
  <figcaption>배포 후 모니터링 지표</figcaption>
</figure>

## 해결 완료

* **빌드 시간 단축**: 평균 10분 → 6분 (−40%)
* **CPU 사용률 안정화**: 최대 100% → 최대 70% 이하
* **Free Tier 유지**: EC2·S3 사용량 한도 내 유지
* **비용 제로**: 추가 과금 없이 CI/CD 환경 운영 성공

## 회고 & 개선 아이디어

* **회고**: 최소한의 최적화만으로 안정적인 파이프라인 구현 가능
* **개선 아이디어**:

    1. EC2 자동 스케일링 (Lambda 스케줄러)
    2. CloudWatch 알람 → Slack 실시간 통보
    3. ECR 레이어 캐시 활성화로 빌드 속도 추가 개선

> OSID CI/CD 최적화 과정을 5분 브리핑 형태로 요약했습니다.



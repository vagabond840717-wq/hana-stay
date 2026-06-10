# HANA STAY — 프로젝트 개요

## 프로젝트 목적

단기 숙박(에어비앤비, 부킹닷컴 등) 다중 객실 운영을 위한 관리 도구 모음.  
iCal 연동을 통해 여러 플랫폼의 예약을 한 화면에서 통합 조회하고,  
청소 스케줄 관리, 도어락 비밀번호 기록, 요금 역산 등을 지원한다.

## 앱 구성 (4개)

| 앱 | 파일 | 목적 | 상태 |
|----|------|------|------|
| 청소 스케줄 | `app/JnJ/index.html` | 체크인/아웃 달력, 청소 일정, 비밀번호/메모 | 운영 중 |
| 예약현황 | `app/JnJ booking/index.html` | 예약 현황 달력, 오버부킹 감지, 알림 | 운영 중 |
| 요금 계산기 | `app/JnJ Price/index.html` | 플랫폼별 수수료 역산 계산기 | 운영 중 |
| 주차 관리 | `app/parking-main/` | 주차 정보 KV 저장 (Cloudflare Worker) | 운영 중 |

## 지원 플랫폼

| 아이콘 | 플랫폼 | 연동 방식 |
|--------|--------|----------|
| ✈ | Airbnb | iCal (webcal://) |
| 🏨 | Booking.com | iCal (webcal://) |
| 🌐 | Trip.com | iCal (webcal://) |
| 🏡 | 리브애니웨어 | iCal (webcal://) |

## 기술 스택

- **프론트엔드**: 순수 HTML/CSS/JS (프레임워크 없음, 단일 파일 앱)
- **백엔드**: Cloudflare Workers (iCal 프록시 + KV 저장소)
- **저장소**: Cloudflare KV (서버) + localStorage (클라이언트 폴백)
- **폰트**: DM Sans, DM Mono (Google Fonts)
- **호스팅**: 별도 (각 index.html 직접 열거나 정적 호스팅)

## 개발 맥락

- 단일 HTML 파일 구조 — 별도 빌드 없이 파일 열기만 하면 동작
- 기능을 하나의 파일에 계속 추가하다 보니 코드 꼬임 발생
- 이 문서들은 리팩토링/신규 기능 추가 시 설계 기준으로 활용

## 디렉토리 구조

```
E:\airbnb\
├── CLAUDE.md                    ← AI 작업 참조용 컨텍스트
├── docs/
│   ├── 01-overview.md           ← 이 파일
│   ├── 02-architecture.md       ← 시스템 아키텍처
│   ├── 03-data-model.md         ← 데이터 구조
│   ├── 04-apps-spec.md          ← 앱별 기능 명세
│   └── 05-known-issues.md       ← 알려진 문제 및 주의사항
└── app/
    ├── JnJ/index.html
    ├── JnJ booking/index.html
    ├── JnJ Price/index.html
    └── parking-main/
        ├── parking-main/worker.js
        └── parking-main/wrangler.toml
```

# Stock ATH Tracker — Design Spec
**Date:** 2026-06-22

## Goal
단일 HTML 파일(`stock_tracker.html`)로, 사용자가 등록한 미국 주식 종목의 현재가와 역대 최고가(ATH) 대비 등락률을 시각적으로 보여주는 정적 웹페이지.

## Constraints
- 백엔드 없음 — 기존 프로젝트가 정적 HTML 파일 기반
- API 키 불필요 — Yahoo Finance 비공식 API 사용
- 외부 라이브러리 없음 — 순수 HTML/CSS/JS

## Data Source
**Yahoo Finance v8 Chart API (비공식)**
- 엔드포인트: `https://query1.finance.yahoo.com/v8/finance/chart/{symbol}?interval=1mo&range=max`
- 반환 데이터: 상장 이후 전체 월봉 (timestamp, high, low, open, close)
- 현재가: `meta.regularMarketPrice`
- ATH 계산: `max(indicators.quote[0].high)` (null 필터링 후)
- CORS 실패 시 폴백: `https://api.allorigins.win/raw?url={encodedUrl}`

## ATH 계산 방식
- 월봉 데이터의 `high` 배열 전체를 순회하여 최댓값 추출
- 월봉 기준이므로 실제 ATH와 최대 1개월 오차 가능 (허용 범위)
- ATH 달성 월의 timestamp → 날짜 표시

## UI 구조

### 레이아웃
```
Header (제목 + 부제목)
Add Section (입력창 + 추가 버튼)
Controls (종목 수 + 새로고침 버튼)
Stock Grid (카드 반응형 그리드, min 320px)
Last Updated (타임스탬프)
```

### 카드 구성 요소
- 티커 + 회사명 + 삭제 버튼
- 현재가 (크게) + ATH 대비 % 뱃지
  - 음수: 빨간색 배경
  - 양수: 초록색 배경
- ATH 가격 + ATH 달성 날짜
- 진행 바: `현재가 / ATH × 100%` 너비

### 상태 처리
- **로딩**: 스피너 표시
- **에러**: 빨간 카드 + 오류 메시지
- **빈 상태**: 안내 문구

## 데이터 흐름
1. 페이지 로드 → localStorage에서 watchlist 복원
2. 각 종목별 Yahoo Finance API 호출 (병렬)
3. ATH + 현재가 계산 → 카드 렌더링
4. 5분마다 자동 새로고침
5. 종목 추가/삭제 → localStorage 즉시 저장

## 파일 구조
```
stock_tracker.html   ← 단일 파일 (styles + scripts 인라인)
```

## 성공 기준
- 종목 추가 후 카드가 정상 렌더링됨
- ATH 대비 % 값이 합리적 (AAPL, NVDA 등으로 검증)
- 페이지 새로고침 후 종목 목록 유지
- 잘못된 티커 입력 시 에러 카드 표시

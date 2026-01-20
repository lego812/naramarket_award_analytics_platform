# Jodalchung Frontend

입찰 낙찰 검색/분석을 위한 Vue 3 + Vite 프론트엔드입니다. 백엔드 API로
낙찰 기록을 조회하고 필터링, 페이지네이션, 업체별 요약을 제공합니다.

## 주요 기능

- 조회 기간/수요기관/키워드 필터
- 서버 조회 + 클라이언트 필터링/페이지네이션
- 업체별 낙찰 건수 및 주요 수요기관 TOP3 요약

## 요구 사항

- Node.js 18+
- 백엔드: `http://localhost:8080`

## 실행 방법

```sh
npm install
npm run dev
```

브라우저에서 `http://localhost:5173` 접속.

## API 호출

프론트는 다음 형태로 GET 요청을 보냅니다:

```
GET /award/results/range?inqryBgnDt=YYYYMMDD0000&inqryEndDt=YYYYMMDD2359&dminsttNm=...&bidNtceNm=...
```

응답 예시:

```
{
  "resultCode": "00",
  "resultMsg": "OK",
  "numOfRows": 0,
  "pageNo": 1,
  "totalCount": 0,
  "items": [ ... ]
}
```

## 프록시 설정

CORS 회피를 위해 Vite 프록시를 사용합니다:

```
/award -> http://localhost:8080
```

자세한 설정은 `vite.config.js`에서 확인하세요.

## 빌드

```sh
npm run build
```

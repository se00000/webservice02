# 웹서비스설계 9주차 실습: 대용량 검색·정렬·페이징

- 학번 / 이름: 202422474 / 서준원
- 제공된 예제(1,000건 인메모리 게시글)를 실행해 기능을 검증하고, 부족한 부분을 고친 결과물입니다.
- 실습 보고서와 테스트 캡처는 LMS에 따로 제출했습니다.

## 실행 방법

| 항목 | 버전 |
|---|---|
| 운영체제 | Windows 11 Home |
| Node.js | v22.23.2 |
| Express | 5.3.0 |
| 브라우저 | Microsoft Edge |

```sh
git clone https://github.com/se00000/webservice02.git
cd webservice02
npm install
node server.js
```

- 접속 주소: http://localhost:3000
- 정상 실행 확인: 콘솔에 `🚀 웹서비스설계 9주차 3학년 전공 실습 서버 구동 완료`가 나오고, 브라우저에 게시판과 오른쪽 검사기 화면이 보입니다.
- 종료: 서버를 실행한 터미널에서 Ctrl + C
- 데이터는 서버를 켤 때마다 새로 만들어집니다. 조회수는 무작위라서 실행할 때마다 달라집니다.

## 파일 구성

```text
.
├── server.js      # Express API 서버 (수정본)
├── index.html     # 게시판 화면과 요청·SQL·JSON 검사기 (수정본)
├── package.json
├── original/      # 제공된 원본 예제 (비교용, 수정하지 않음)
│   ├── server.js
│   ├── index.html
│   └── README.md
└── README.md
```

원본과 수정본의 차이는 아래 명령으로 볼 수 있습니다.

```sh
git diff --no-index original/server.js server.js
git diff --no-index original/index.html index.html
```

## API

| 메서드 | 경로 | 설명 |
|---|---|---|
| GET | `/api/posts?page=1&limit=10&sort=latest&search=` | 오프셋 페이징. sort는 `latest`, `oldest`, `views`만 허용하고 그 외는 `latest` |
| GET | `/api/posts/cursor?limit=10&cursor=&search=` | 커서 페이징. id 내림차순 고정, `limit + 1` 조회로 `hasNext` 판단 |

## 원본에서 수정한 내용

| 번호 | 파일 | 기존 동작 | 수정 후 동작 |
|---|---|---|---|
| 1 | `server.js` 85줄 | `sort=__proto__`는 500 에러, `sort=constructor`는 latest가 아닌 엉뚱한 정렬 | `Object.hasOwn`으로 직접 정의한 정렬 키만 허용, 나머지는 latest |
| 2 | `server.js` 124~130줄 | 커서 API가 `search`를 무시하고 전체 목록 반환 | 제목·본문 검색 → `id < cursor` → `limit + 1` 순서로 처리 |
| 3 | `index.html` 349줄 | 커서 모드 요청에 검색어가 빠짐 | 검색어가 있으면 요청에 `search`를 함께 보냄 |
| 4 | `index.html` 221, 315줄 | 커서 모드에서 정렬을 바꿔도 결과가 그대로 | 커서 모드에서 정렬 선택 비활성화, 탭에 "id DESC 고정" 표시 |
| 5 | `index.html` popstate, DOMContentLoaded | 주소에 `?sort=abc`가 있으면 정렬 선택창이 빈칸 | 허용된 값이 아니면 latest로 바꿔 선택창과 결과 일치 |

## 테스트 결과 요약

평가 항목별 29개 테스트를 브라우저에서 실행했고, 모두 기대 결과와 일치했습니다. 입력, 기대·실제 결과, 캡처는 보고서에 있습니다.

- 검색·정렬: 제목·본문 검색, 결과 없음, 세 가지 정렬, 조회수 동점 시 id DESC, `views;DROP TABLE--`·`__proto__`·`constructor` → latest
- 페이지네이션: `page=0, -1, abc` → 1페이지, `limit=1000` → 100, 검색·정렬 변경 시 1페이지, 커서 검색 Redis 끝까지 넘겨 83건·중복 0건(오프셋 totalCount 83과 일치), 마지막에 더 보기 비활성화
- 비동기 처리: 5글자 연속 입력 시 요청 1회, 3G 쓰로틀링에서 검색어를 바꾸면 이전 요청 취소와 AbortController 로그, 최종 결과만 표시
- URL 상태 복원: 검색어·정렬·페이지가 주소에 반영되고, 새로고침·뒤로 가기 후 주소·입력창·정렬·결과 일치
- 응답 시간 관찰값: `executionTimeMs` 약 0.1~2ms, `X-Response-Time` 0.94ms (1,000건 인메모리 배열)

## 개념 정리

1. **오프셋과 커서에 적합한 화면**: 오프셋(`LIMIT ? OFFSET ?`)은 페이지 번호로 원하는 페이지에 바로 가야 하는 게시판, 상품 목록에 맞습니다. 커서(`WHERE id < ? ORDER BY id DESC`)는 마지막으로 본 id 다음부터 이어서 불러오므로 SNS 피드, 채팅처럼 더 보기·무한 스크롤 화면에 맞습니다.
2. **디바운스와 요청 취소**: 디바운스는 입력이 300ms 동안 멈췄을 때만 요청해 불필요한 호출을 줄입니다. 요청 취소(AbortController)는 새 요청 전에 이전 요청을 끊어, 늦게 온 이전 응답이 최신 결과를 덮어쓰지 못하게 합니다.
3. **limit + 1 원리**: 10개가 필요하면 11개를 조회합니다. 11개가 오면 `hasNext = true`로 정하고 11번째는 버립니다. `COUNT(*)` 없이 다음 페이지 존재 여부를 알 수 있습니다.
4. **계산식**: `offset = (page − 1) × limit`, `totalPages = max(1, ceil(totalCount / limit))`. 예를 들어 page 3, limit 10이면 offset 20이고, totalCount 83이면 totalPages 9입니다.

## AI 활용

- 사용 도구: Claude Code (Claude Opus 5.5)
- 사용한 부분: 원본 예제의 문제점 탐색, JavaScript 문법 설명, 수정 코드 예시 제안, 수정 코드 실행 점검, 테스트·캡처 방법 정리, 보고서 초안, 이 저장소의 폴더 구성과 README 작성
- 직접 한 것과 검증한 내용: `server.js`, `index.html`의 수정 코드는 직접 입력했고, 29개 테스트를 직접 브라우저에서 실행·캡처했습니다. 커서 검색 수정 중 둘째 페이지부터 검색이 풀리는 오류를 찾아 고친 뒤 83건·중복 0건을 확인했습니다.

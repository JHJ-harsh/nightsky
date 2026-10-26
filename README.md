# 단국네컷 (DKSH NOID FOUR CUT)

키오스크에서 사진 6장을 찍고, 그중 4장을 골라 프레임에 합성한 뒤 바로 인쇄하는 **네컷 사진 부스** 프로젝트입니다.
합성된 사진은 선택적으로 웹 **갤러리**(좋아요/랭킹)에 올릴 수 있습니다.

| 폴더 | 역할 | 스택 |
| --- | --- | --- |
| [noid_frontend/](noid_frontend/) | 키오스크 촬영 UI + 이미지 합성/인쇄 API + (포팅된) 갤러리 | Next.js 16 (Pages Router), React 19, SQLite, sharp |
| [noid_backend/](noid_backend/) | 외부 공개용 사진 갤러리 서버 (업로드 수신, 목록, 좋아요, 다운로드) | Flask 3, SQLAlchemy 2, SQLite, Jinja2 |

> 현재 프론트엔드의 외부 갤러리 업로드와 QR 생성 코드는 주석 처리되어 있습니다. 즉 **지금은 키오스크(프론트엔드) 단독으로 촬영 → 합성 → 인쇄까지 동작**하고, Flask 백엔드와는 연결되어 있지 않습니다. 갤러리 기능은 `pages/gallery/*` 로 Next.js 안에 포팅되어 있습니다.

---

## 전체 구조

```
nightsky/
├── noid_backend/                  # Flask 갤러리 서버
│   ├── app.py                     # 모든 라우트 (단일 파일)
│   ├── utils/models.py            # SQLAlchemy Photo 모델
│   ├── templates/                 # Jinja2 템플릿
│   │   ├── index.html             #   전체 갤러리 (최신순, 10장/페이지)
│   │   ├── likes.html             #   좋아요 랭킹 (12장/페이지)
│   │   ├── photo.html             #   단일 사진 + 좋아요 버튼 (팝업 embed 대상)
│   │   └── download.html          #   QR 스캔 후 다운로드 페이지
│   ├── static/                    # 배경 SVG, download.css
│   ├── images/                    # 업로드된 합성 사진 ({session}.png)
│   ├── db.db.zip, images.zip      # DB/이미지 백업 아카이브
│   └── requirements.txt
│
└── noid_frontend/                 # Next.js 키오스크 앱
    ├── pages/
    │   ├── _app.js, _document.js  # 전역 CSS(vars.css, default.css) 로드
    │   ├── index.js               # 시작 화면 → /frame
    │   ├── frame.js               # 프레임 카테고리 선택 (NOID / 박람회 / 자율)
    │   ├── noid_select.js         #   NOID 프레임 목록
    │   ├── noid_dankook_b.js      #     NOID BLACK 미리보기 (+인쇄 매수)
    │   ├── noid_dankook_n.js      #     NOID NAVY 미리보기 (+인쇄 매수)
    │   ├── ddp_select.js          #   서울진로직업박람회 프레임 목록
    │   ├── ddp_dwyl.js            #     sjj_2 프레임 미리보기
    │   ├── ddp_seoul.js           #     sjj_1 프레임 미리보기
    │   ├── n_select.js            #   자율 프레임 목록 (blue0 / blue2)
    │   ├── n_blue.js              #     자율 프레임 미리보기 (?frame=)
    │   ├── take_photo-1.js        # 카메라 미리보기 (촬영 준비)
    │   ├── take_photo-2.js        # 세션 생성 + 6장 자동 연속 촬영
    │   ├── take_select_photo.js   # 6장 중 4장 선택 → 업로드
    │   ├── download.js            # 결과 사진 표시, 30초 후 홈으로
    │   ├── gallery/               # Flask 갤러리의 Next.js 포팅 (SSR)
    │   │   ├── index.js           #   최신순 목록
    │   │   ├── likes.js           #   좋아요 랭킹
    │   │   ├── photo.js           #   단일 사진 + 좋아요
    │   │   └── download.js        #   다운로드 페이지
    │   └── api/
    │       ├── session.js         # POST  촬영 세션 생성
    │       ├── upload.js          # POST  4장 합성 + PNG 저장 + 자동 인쇄
    │       ├── download.js        # GET   세션 확인 (+QR)
    │       ├── like.js            # POST  좋아요 +1
    │       └── gallery/
    │           ├── get_image.js       # GET 이미지 인라인
    │           └── download_image.js  # GET 이미지 첨부 다운로드
    ├── components/
    │   └── CopiesSelector.js      # 인쇄 매수(1~4) 선택 위젯 → localStorage
    ├── lib/
    │   ├── camera.js              # getUserMedia 래퍼, 에러 메시지 한글화
    │   ├── printer.js             # 프린터 자동 출력 (CUPS lp / Windows PowerShell)
    │   └── galleryDb.js           # photos 테이블 SQLite 헬퍼
    ├── styles/                    # 페이지별 CSS Module (Figma export 기반)
    └── public/
        ├── images/                # 프레임 PNG (dankook_b, dankook_n, sjj_1, sjj_2, blue0, blue2 …)
        ├── models/                # face-api.js 모델 (tiny face detector, landmark 68)
        └── static/                # 갤러리 배경 SVG
```

---

## 사용자 흐름 (키오스크)

```
index ──▶ frame ──┬─▶ noid_select ──▶ noid_dankook_b / noid_dankook_n ─┐
                  ├─▶ ddp_select  ──▶ ddp_dwyl / ddp_seoul ───────────┤  (인쇄 매수 선택)
                  └─▶ n_select    ──▶ n_blue?frame=blue0|blue2 ───────┤
                                                                      ▼
                                          take_photo-1?frame=<프레임 이름>   카메라 미리보기
                                                                      ▼
                                          take_photo-2?frame=<프레임 이름>   POST /api/session
                                                  5초 대기 후 8초 간격 × 6장 자동 촬영
                                                                      ▼
                                          take_select_photo                 4장 선택 → POST /api/upload
                                                                      ▼
                                          download?session=<id>             결과 표시, 30초 후 / 로
```

| 프레임 이름(`frame`) | 프레임 파일 | 선택 페이지 |
| --- | --- | --- |
| `dankook_b` | `public/images/dankook_b.png` | NOID BLACK |
| `dankook_n` | `public/images/dankook_n.png` | NOID NAVY |
| `sjj_1`, `sjj_2` | `public/images/sjj_*.png` | 2026 서울진로직업박람회 |
| `blue0`, `blue2` | `public/images/blue*.png` | 자율 프레임 |

### 페이지 간 데이터 전달 (localStorage)

| 키 | 쓰는 곳 | 읽는 곳 | 내용 |
| --- | --- | --- | --- |
| `printCopies` | `CopiesSelector` | `take_select_photo` → upload | 인쇄 매수 1~4 |
| `session` | `take_photo-2` | `take_select_photo` | `/api/session` 이 만든 UUID |
| `capturedPhotos` | `take_photo-2` | `take_select_photo` | 촬영본 6장 (JPEG data URL, 10:13 비율, 좌우 반전) |

---

## 프론트엔드 API (Next.js `pages/api`)

| 메서드 | 경로 | 입력 | 동작 |
| --- | --- | --- | --- |
| POST | `/api/session` | JSON `{ frame }` | `photo_sessions` 에 새 UUID 행 생성 → `{ sessionID }` |
| POST | `/api/upload` | multipart: `session`, `copies`, `upload`, `photo1`~`photo4` | 이미지 검증 → 프레임에 4장 합성 → 1200×1800px(4×6in, 300dpi)로 정규화 → `images/{session}.png` 저장 → `printPhoto()` 비동기 호출 → DB `photofile` 갱신 |
| GET | `/api/download?session=` | | 세션 존재 확인, `qr_code` 반환 (현재 QR은 null) |
| POST | `/api/like` | JSON `{ session }` | `photos.like` +1 |
| GET | `/api/gallery/get_image?session=` | | `images/{session}.png` 인라인 스트리밍 |
| GET | `/api/gallery/download_image?session=` | | 같은 파일을 첨부(attachment)로 전송 |

### 이미지 합성 (`upload.js` → `createImage`)

- 각 사진을 **508×668** 로 리사이즈해 프레임 위 고정 좌표에 배치합니다.
  `(74,355)`, `(619,92)`, `(74,1053)`, `(619,790)` — 좌/우 두 열이 엇갈린 레이아웃.
- 합성 결과를 `fit: "contain"` 으로 1200×1800 흰 배경에 맞춰 잘림 없이 인쇄되게 합니다.

### 자동 인쇄 (`lib/printer.js`)

- macOS/Linux: CUPS `lp` (4×6 광택지, 여백 0, fit-to-page)
- Windows: 임시 `.ps1` (UTF-8 BOM) + .NET `System.Drawing.Printing` 으로 4×6 용지 중앙 출력
- 실패해도 throw 하지 않으며 업로드 응답을 막지 않습니다(fire-and-forget).
- 환경변수 (`.env.local`):

| 변수 | 기본값 | 설명 |
| --- | --- | --- |
| `PRINT_ENABLED` | `true` | `false` 면 인쇄 안 함 |
| `PRINT_DRYRUN` | `false` | `true` 면 명령만 로그 |
| `PRINT_PRINTER` | `EPSON L18050 Series` | 프린터 이름 |
| `PRINT_COPIES` | `1` | 기본 매수 (요청의 `copies` 가 우선) |
| `PRINT_MEDIA` / `PRINT_MEDIA_TYPE` | `Custom.102x152mm` / `photographic-glossy` | CUPS 전용 |
| `PRINT_FIT` | `true` | 용지에 맞춤 |
| `PRINT_ORIENTATION` | `portrait` | `portrait` / `landscape` |
| `PRINT_EXTRA` | | `lp` 에 그대로 넘길 추가 옵션 |

---

## 백엔드 API (Flask `app.py`, 포트 8000)

| 메서드 | 경로 | 동작 |
| --- | --- | --- |
| GET | `/?page=` | `upload=True` 사진을 최신순 10장씩 → `index.html` |
| GET | `/likes?page=` | 좋아요 많은 순 12장씩 → `likes.html` |
| GET | `/photo?session=` | 단일 사진 + 좋아요 → `photo.html` |
| POST | `/upload?password=` | multipart `file`(`{session}.png`), `upload` → `images/` 저장 + `Photo` 행 생성. 비밀번호 불일치 시 403 |
| POST | `/api/like` | JSON `{ session }` → 좋아요 +1 |
| GET | `/download?session=` | 다운로드 페이지 → `download.html` |
| GET | `/download_image?session=` | 첨부 파일로 전송 |
| GET | `/get_image?session=` | 이미지 인라인 전송 |

갤러리 팝업은 목록 페이지가 `<embed src="/photo?session=...">` 로 단일 사진 페이지를 띄우는 방식이며, 하단 시트를 아래로 150px 이상 끌면 닫힙니다. Next.js `pages/gallery/*` 도 같은 방식입니다.

---

## 데이터베이스 (SQLite `db.db`)

각 서버의 실행 디렉터리(`process.cwd()` / Flask 작업 디렉터리)에 `db.db` 가 생성됩니다.

**`photo_sessions`** — 프론트엔드 키오스크 전용

| 컬럼 | 타입 | 설명 |
| --- | --- | --- |
| `id` | TEXT PK | 세션 UUID |
| `frame` | TEXT | 프레임 이름 |
| `qrfile` | TEXT | QR data URL (현재 미사용) |
| `photofile` | TEXT | 합성 이미지 base64 |

**`photos`** — 갤러리 (Flask `utils/models.py` 와 `lib/galleryDb.js` 가 같은 스키마)

| 컬럼 | 타입 | 설명 |
| --- | --- | --- |
| `id` | TEXT PK | 세션 UUID (= 이미지 파일명) |
| `photoUrl` | TEXT | 이미지 URL |
| `createdAt` | DATETIME | 업로드 시각 (UTC) |
| `like` | INTEGER | 좋아요 수 |
| `upload` | BOOLEAN | 갤러리 공개 여부 |

이미지 파일은 DB가 아닌 `images/{session}.png` 에 저장됩니다.

---

## 실행 방법

### 프론트엔드

```bash
cd noid_frontend
npm install
npm install sharp      # upload.js 에서 사용하지만 package.json 에 빠져 있음
npm run dev            # http://localhost:3000
```

카메라(`getUserMedia`)는 **HTTPS 또는 `localhost`** 에서만 동작합니다. 키오스크 기기에서는 `http://localhost:3000` 으로 접속하세요.

### 백엔드

```bash
cd noid_backend
python -m venv venv && source venv/bin/activate   # Windows: venv\Scripts\activate
pip install -r requirements.txt
python app.py          # http://localhost:8000
```

---

## 알아둘 점 / 알려진 문제

- **프레임 경로**: `upload.js` 의 `createImage` 가 프레임을 `path.join(process.cwd(), "../noid/public/images")` 에서 읽습니다. 폴더 이름이 `noid_frontend` 이면 프레임 파일을 찾지 못하므로 `public/images` 로 고치거나 폴더 이름을 `noid` 로 맞춰야 합니다.
- **sharp 의존성 누락**: `package.json` 에 `sharp` 가 없습니다.
- **미사용 의존성**: `axios`, `form-data`, `qrcode`, `node-fetch`, `react-snowfall`, `face-api.js` (및 `public/models/`) 는 현재 코드에서 쓰이지 않거나 주석 처리되어 있습니다. 백엔드 `requirements.txt` 에도 selenium, pandas, scikit-learn 등 `app.py` 와 무관한 패키지가 많습니다.
- **없는 페이지 참조**: Flask 는 `forbidden.html` 템플릿을, `pages/download.js` 는 `/forbidden` 경로를 사용하지만 둘 다 존재하지 않습니다.
- **하드코딩된 비밀값**: Flask `/upload` 의 비밀번호가 `app.py` 와 `upload.js` 주석에 평문으로 들어 있습니다. 공개 저장소에 올리기 전에 환경변수로 옮기세요.
- **세션 값 검증 없음**: `session` 쿼리값을 그대로 파일 경로에 사용합니다(`./images/{session}.png`). 외부 공개 시 UUID 형식 검증을 권장합니다.
- Flask 는 모듈 전역 SQLAlchemy 세션 하나를 모든 요청에서 공유합니다.
- `styles/vars.css` 는 Figma export 템플릿으로 비어 있습니다.

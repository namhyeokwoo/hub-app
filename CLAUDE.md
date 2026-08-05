# 크로스 프로젝트 허브 원격 제어 앱 (hub-app)

`discord_bot.py`가 제공하던 DART/IT뉴스/논문브리핑 원격 트리거를 안드로이드 앱 + 웹으로 확장한
프로젝트. 로컬(Tailscale) 모드와 클라우드(GitHub Actions) 모드를 토글해서 쓸 수 있고, GitHub
Pages로 웹 버전도 배포됨. `namhyeokwoo/hub-app`(public — GitHub Pages 무료 배포를 위해 필요,
전체 히스토리 시크릿 스캔 후 전환함).

## 구조

```
로컬 모드:
  안드로이드 앱/웹 → fetch() + Bearer <HUB_API_TOKEN>
    → Tailscale mesh (PC "pc" + 폰 "s23-ultra")
      → K:\claude\hub_api.py (Flask :8787, Tailscale IP 전용 바인딩)
        → K:\claude\hub_core.py (공유 실행/파싱 모듈)

클라우드 모드:
  안드로이드 앱/웹 → GitHub PAT로 workflow_dispatch → 각 repo의 .github/workflows/run.yml
    → 결과를 api.github.com Contents API로 조회 (raw.githubusercontent.com 아님, 아래 CORS 참고)
```

- `www/index.html` — 앱 본체(Capacitor, webDir=www, 완전 번들형 SPA, 외부 CDN 의존 없음).
  설정화면(IP/포트/토큰 또는 GitHub PAT, localStorage 저장) + 홈 대시보드(타일) + 상세화면(리포트
  뷰어) 3화면
- `CLOUD_JOBS` / `PROJECTS_STATIC` — 클라우드 모드에서 사용하는 job 정의와 정적 프로젝트 목록.
  **로컬 모드는 `K:\claude\projects.json`을 따로 읽으므로, 새 프로젝트 타일 추가 시 두 곳 다 고쳐야 함**
- `mdToHtml()` — 외부 CDN 없이 자체 구현한 offline 마크다운 파서(#/##/###, 표, **bold**, 링크,
  1단 중첩 불릿, `<details>/<summary>` 통과 지원)

## 등록된 job (dart/news/paper/ai_bids/influencer, 5개)

새 프로젝트를 hub에 연결하려면 `hub_core.py`에 `JobResult`/`run_*`/`parse_*` 함수 쌍 패턴을
그대로 복제. `hub_api.py`에 `/run/{job}`, `/report/{job}` 라우트 추가. `www/index.html`의
`JOB_TIMEOUT`/`JOB_LABEL_RUNNING`/`CLOUD_JOBS`/`PROJECTS_STATIC`에 항목 추가.
실행시간이 긴 job(예: ai_bids 10~20분)은 `CLOUD_JOBS[job].pollDeadlineMs`로 폴링 타임아웃 개별
조정 가능(기본 8분, ai_bids는 25분).

## 공개 리포트 페이지 (`www/report/`, 2026-08-06)

기존 제어 대시보드(`www/index.html`, PAT 입력·실행 버튼·FCM 토큰 등 포함)와 완전히 분리된
읽기 전용 정적 페이지. `www/report/index.html`은 자체 인증/localStorage 없이
`www/report/data/{dart.md,news.html,paper.html}` 3개 파일만 fetch해서 보여준다.

- **대상 3개(dart 공시, it-news, ai-weekly-paper-briefing)만 포함**. ai-public(AI·SW
  조달, 나라장터 API 401로 아직 한 번도 성공 실행 못함)과 x-influencer-briefing(X API 토큰
  미발급)은 아직 안정화 전이라 의도적으로 제외 — 두 project는 `www/report/`의 어떤 파일에도
  등장하지 않음(빈 섹션조차 없음, 완전 부재)
- **데이터 흐름**: dart/it-news/ai-weekly-paper-briefing(모두 private repo) 각각의
  `.github/workflows/run.yml` 마지막 스텝이 결과 파일을 `www/report/data/`에 직접 커밋·push함
  (`git clone` + `HUB_APP_TOKEN` 시크릿). 페이지 로드 시점엔 어떤 API도 호출하지 않음 — 순수
  정적 파일 서빙이라 private repo 자격증명이 클라이언트에 노출될 일 자체가 없음
  (hub-app이 public repo이므로 브라우저에서 직접 그 repo들에 접근할 방법이 없기 때문)
- **`HUB_APP_TOKEN`**: fine-grained PAT, **hub-app repo에만** Contents(Read and write) 권한.
  dart/it-news/ai-weekly-paper-briefing 3개 repo의 GitHub Secret으로 각각 등록 필요(사용자가
  직접 발급 — 위 FCM과 동일하게 Claude 자동모드는 크리덴셜 직접 취급 안 함). 안드로이드
  앱이 쓰는 기존 PAT(Actions+Contents write, 5개 repo 대상)와는 **별개의 좁은 권한 토큰** —
  재사용 금지(유출 시 피해 범위를 hub-app repo 하나로 제한하기 위함)
- `www/robots.txt`(`Disallow: /report/`) + 페이지 자체 `<meta name="robots" content="noindex,
  nofollow">` 이중으로 검색엔진 색인 차단
- `pages.yml`은 `www/**` 전체를 배포하므로 `www/report/`도 별도 워크플로 수정 없이 자동 포함됨

## ZD캘린더 타일 (embedUrl 패턴)

`embedUrl` 필드가 있는 타일(zdnet-event-app)은 job 실행 없이 `openEmbed()`로 곧바로
`iframe.src`를 여는 별도 경로 — 기존 "실행→결과보기" job 흐름과 다름. 알려진 한계: ZD 앱 자체의
설정모달/인증/배너가 iframe 안에 그대로 노출되는 이중 UI(의도적으로 감수, 필요시 postMessage
passthrough로 개선 가능).

## FCM 푸시 알림

`firebase-admin`(서버측, dart/it-news/zdnet-event-app repo에 `notify_fcm.py`) +
`@capacitor/push-notifications`(클라이언트측). 시크릿 없으면 조용히 스킵. Firebase 프로젝트명
`hub-app-692b4`. **비밀키(서비스계정 JSON) 처리는 사용자가 직접 PowerShell로 등록** — Claude
자동모드가 크리덴셜 파일 직접 취급을 차단함. 포그라운드에서 알림이 시스템 트레이에 안 뜨는 건
Android 기본 동작(버그 아님) — `pushNotificationReceived` 핸들러 미구현 상태(의도적 보류).

## 실기기 배포 시 겪은 버그 (브라우저 테스트로는 안 드러남 — 재발 시 우선 확인)

1. Android 9+ cleartext 차단 → `AndroidManifest.xml`에 `usesCleartextTraffic="true"`
   (`cap sync`가 덮어쓰지 않으므로 수동 편집 유지)
2. WebView mixed-content 차단(①만으론 부족) → `capacitor.config.json`에
   `"server": {"androidScheme": "http"}`
3. WebView GET 캐싱 의심 → 서버 `Cache-Control: no-store` + 클라이언트 `fetch(cache:"no-store")` 이중 방어
4. 마크다운 링크 파싱: `[기재정정]` 같은 대괄호 포함 텍스트는 그리디 매칭 필요
5. 결과 조회 CORS: `raw.githubusercontent.com`은 `Authorization` 헤더 preflight(OPTIONS)를
   403으로 거부 → `api.github.com` Contents API(`Accept: application/vnd.github.raw`)로 통일

## 알려진 함정 (재발 방지)

- **PAT 저장소 누락 → 403 아닌 404**: fine-grained PAT의 "Only select repositories"에서
  검색 후 실제로 클릭 추가 안 하면 그 repo는 권한 밖 — private repo라 403 대신 404로 뜸(존재
  자체를 숨김). 새 job/repo 추가 시 PAT 목록에 추가했는지부터 확인
- **웹(GitHub Pages)과 APK는 localStorage가 공유 안 됨** — origin이 다르므로 PAT/토큰을
  각각 따로 입력해야 함. 웹에서 401 나면 "이 origin에 PAT를 넣은 적 있는지"부터 확인
- **PAT Contents 권한 No access → 403**: Actions(dispatch/poll)는 되는데 결과 조회만 403.
  Contents는 Read-only로 충분
- **`android/.idea/*.xml`은 커밋 대상 아님**(머신별 값, gitignore 처리됨) — Android Studio로
  열 때마다 재생성되는 건 정상, 무시할 것

## 주의사항

- `hub_api.py`의 `/run/*`을 실제 토큰으로 회귀 테스트하면 하위 스크립트가 진짜로 실행됨(DART API
  호출, Claude API 호출, 10~20분짜리 전수조사 등) — 인증 없이 401만 확인하거나 hub_core 함수를
  직접 mock할 것
- `android/.gitignore`에 `google-services.json`/`*firebase-adminsdk*.json` 반드시 유지(public
  repo이므로 실수로 커밋되면 즉시 노출)

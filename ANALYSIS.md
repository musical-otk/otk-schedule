# otk-schedule 앱 분석

> 작성일: 2026-07-17 (코드 스냅샷 기준, 최근 커밋: `1307d20 feat: 비더슈탄트 프리셋 추가, 사진 다운로드, 달력 레이아웃 수정`)

## 1. 앱의 정체

`otk-schedule`은 개별 뮤지컬 관극 앱(roger2026 계열, 예: eoduk2026, dive2026 등)과는 **다른 종류의 앱**이다.
여러 관극 앱의 스케줄을 한 화면(달력)에 모아 보여주고, 각 앱을 탭으로 전환해 iframe으로 띄워주는 **허브(포털) 앱**이다.

- 파일 구성: `index.html`(단일 파일, HTML+CSS+JS 전부 포함) + `manifest.json` + `sw.js` + 아이콘 4개
- 빌드 도구 없음, 프레임워크 없음 — 순수 Vanilla JS
- PWA (홈 화면 설치 가능, 서비스워커로 오프라인 캐싱)
- 배포: GitHub Pages 추정 (`start_url: /otk-schedule/`, repo `musical-otk/otk-schedule`)

## 2. 핵심 개념: "등록된 앱(apps)"

사용자가 설정에서 여러 "앱"을 등록하면, 각 앱은 다음 정보를 가진다 (`hub_apps` localStorage에 저장):

```js
{
  id,            // 'app_' + timestamp 또는 프리셋 고정 id
  name,          // 표시 이름 (예: "어둑시니")
  icon,          // 이모지 or 1~2글자 (선택)
  scheduleUrl,   // 해당 앱의 schedule.json URL — 달력에 스케줄 표시용
  appUrl,        // 해당 앱의 실제 URL — 탭 클릭 시 iframe으로 열림
  color,         // 배지/탭 색상 (hex)
  recordsKey,    // 해당 앱이 localStorage에 저장하는 관람기록 키 (선택)
}
```

### 프리셋 앱 (`PRESET_APPS`, index.html:551)
현재 하드코딩되어 있는 3개:
| id | name | scheduleUrl | recordsKey |
|---|---|---|---|
| app_outcasts | 아웃캐스트 | outcasts2026/schedule.json | outcasts_records |
| app_eoduk | 어둑시니 | sinimuni2026/schedule.json | eoduk_records |
| app_widerstand | 비더슈탄트 | widerstand2026/schedule.json | wider_records |

> ⚠️ CLAUDE.md의 "생성된 앱 목록"에는 `dive2026`(다이브 2026)도 있는데, `PRESET_APPS`에는 아직 없다. 새 앱을 만들 때마다 이 프리셋 목록에 추가해줘야 허브에서 원클릭으로 등록 가능하다.

## 3. 데이터 소스와 흐름

### 3.1 스케줄 (외부 fetch, 읽기 전용)
- `init()` 시점에 등록된 모든 앱의 `scheduleUrl`을 병렬로 `fetch()` → 실패해도 무시(catch 무시, 조용히 스킵)
- 앱을 새로 추가할 때도 그 자리에서 fetch
- 배우/캐스트 필드는 스키마 없이 **자동 감지**: `date`, `time`, `_`로 시작하는 키를 제외한 나머지 키를 전부 "배우 필드"로 간주 (`getActorFields`)

### 3.2 관람 기록 (다른 앱의 localStorage를 직접 읽음)
- `getAttended()`가 등록된 각 앱의 `recordsKey`로 **동일 출처(origin)의 localStorage를 직접 읽어** 관람 여부/좌석 정보를 가져옴
- 즉, 개별 관극 앱들이 GitHub Pages의 **같은 도메인**(`musical-otk.github.io`) 하위 경로에 있어야 localStorage 공유가 성립 (동일 출처 정책)
- iframe으로 실제 앱을 열 때 이 값이 쓰이는 게 아니라, **브라우저의 공유 localStorage**를 그대로 읽는 구조라서 별도 API/서버 통신 없이 동작

### 3.3 자체 데이터 (otk-schedule 고유)
| localStorage 키 | 내용 |
|---|---|
| `hub_apps` | 등록된 앱 목록 |
| `hub_prefs` | 앱별 선택된 배우 필터 `{appId: [name,...]}` |
| `hub_theme` | 'dark' \| 'light' |
| `hub_custom` | 기타(수동) 일정 목록 |
| `hub_notes` | 세션별 메모/사진 `{`\`appId__date__time\``: {memo, photoIds}}` |

### 3.4 사진 저장 (IndexedDB)
- `PhotoDB` (index.html:1135) — IndexedDB `otk-photos` DB, `photos` object store, autoIncrement key → base64 dataURL 값
- localStorage에는 photoIds(숫자 배열)만 저장하고, 실제 이미지 바이너리는 IndexedDB에 저장 (localStorage 용량 한계 회피)
- 구버전 데이터(예전엔 base64를 localStorage에 직접 저장) → IndexedDB 자동 마이그레이션 로직 존재 (`migratePhotosToIndexedDB`, init 시 1회 실행)
- 업로드 시 `resizeImage()`로 최대 1400×1800, JPEG quality 0.82로 리사이즈 후 저장

## 4. 화면/기능 구조

### 4.1 달력 뷰 (`#view-cal`, 기본 화면)
- 월 단위 그리드, 날짜 셀에 등록된 모든 앱의 세션을 배지로 표시 (앱 색상 + 아이콘/이니셜 + 시간)
- 관람 완료 세션은 `✅` 체크, 중복 관람(같은 세션 2회 이상)은 `✅×N`
- 배우 필터가 걸려있으면 필터에 안 걸리는 세션은 숨김 (단, 실제 관람 기록이 있으면 필터와 무관하게 노출 — index.html:917)
- FAB(+) 버튼으로 기타 일정 빠른 추가

### 4.2 날짜 상세 시트 (`openDay`)
- 해당 날짜의 모든 앱 세션 + 기타 일정을 시간순 정렬해서 표시
- 관람 기록이 있으면 좌석 정보(구역/열/번) 표시
- 세션별로 메모/사진 추가 가능 (`note-row`)

### 4.3 앱 탭 전환 (`switchView`)
- 상단 탭 클릭 → 달력이 숨겨지고 해당 앱의 `appUrl`을 iframe(`about:blank` → 최초 클릭 시 lazy load)으로 표시
- 탭을 다시 누르면 달력으로 복귀 (토글)
- iframe은 앱마다 하나씩 유지되어 있어 탭 전환 시 상태 유지됨 (재로드 안 됨)

### 4.4 설정 시트
- 등록된 앱 목록 관리 (추가/삭제)
- 앱 추가: 프리셋에서 원클릭 추가 OR 직접 입력 (이름/아이콘/scheduleUrl/appUrl/recordsKey/색상)
- 앱별 배우 필터 칩 (`extractActors`로 스케줄에서 자동 추출)

### 4.5 기타 일정 (Custom Event)
- 어떤 앱에도 속하지 않는 수동 일정 (날짜/시간/이름/메모/색상/사진)
- `hub_custom`에 저장, CRUD 전체 지원 (수정/삭제 가능 — 앱 세션과 달리 사진도 삭제 시 IndexedDB에서 정리)

### 4.6 사진 뷰어
- 핀치 줌(터치 2점), 팬(드래그), 더블탭 줌 토글, 스와이프 없이 좌우 버튼 네비게이션
- 다운로드 버튼 (`pvDownload`, `<a download>` 방식)

### 4.7 테마
- 다크/라이트 토글, CSS 변수(`:root` / `body.light`)로 전환, `meta[theme-color]`도 함께 갱신

### 4.8 PWA / 오프라인
- `sw.js`: `schedule.json`은 네트워크 우선(최신 반영) + 실패 시 캐시 폴백, 나머지 정적 자원은 캐시 우선
- 뒤로가기 버튼을 가로채서 열린 오버레이(시트)를 닫는 용도로 사용 — PWA 앱이 뒤로가기로 바로 종료되지 않도록 `history.pushState` 트릭 사용 (index.html:1414-1423)

## 5. 코드 구조 특징

- 전역 함수 + 전역 변수 기반, 모듈/클래스 없음. `onclick="..."` 인라인 핸들러 다수 (HTML과 강하게 결합)
- 상태: `schedules`(fetch된 스케줄 캐시), `currentYear/currentMonth`, `currentView`, 각종 `_custom*`/`_note*`/`_pv*` 임시 변수
- 색상 처리: `COLOR_PALETTE`(9색 고정) + `hexToRgba()`로 배지/칩 스타일 동적 생성
- 렌더링은 전부 `innerHTML` 문자열 조립 방식 (가상 DOM 없음, 매번 풀 리렌더)

## 6. 잠재 개선 포인트 (참고용, 미구현)

- `PRESET_APPS`에 `dive2026` 누락 — 신규 앱 추가 시마다 여기도 수동 업데이트 필요 (누락되기 쉬운 지점)
- `fetch(scheduleUrl)` 실패 시 완전히 무시(`catch {}`) — 사용자에게 실패 피드백 없음 (스케줄 안 보여도 원인 불명)
- iframe 방식이라 등록 앱이 늘어날수록 동시에 여러 iframe이 메모리에 상주 (탭 전환만으로는 unload 안 됨)
- `recordsKey`로 다른 앱의 localStorage를 직접 읽는 구조라, 개별 앱들이 반드시 동일 출처(같은 GitHub Pages 도메인) 하위여야 함 — 이 전제가 코드 어디에도 문서화되어 있지 않음
- 배우 필터가 앱별 독립이라, 특정 배우 조합(페어)을 가로질러 필터링하는 기능은 없음
- 커스텀 이벤트에는 있는 수정/삭제가 "앱 세션 관람기록"에는 없음 (그건 각 개별 앱에서 관리하는 영역이라 의도된 것으로 보임)

## 7. 참고: 보안 메모

- 이 저장소(`otk-schedule`)의 `git remote` URL에 GitHub Personal Access Token이 평문으로 포함되어 있음 (`.git/config`). 로컬 사용에는 문제없지만, 이 config 파일이 백업/공유되면 토큰이 노출된다. 필요시 GitHub CLI(`gh auth`)나 credential helper로 교체하는 걸 권장.

# 66index-creative-app · v2.1

> 오늘의 64괘 — 정감록·예수제자·노자·장자의 길 · 개인 만다라 궤적

주역 64괘를 토대로 그날의 흐름을 읽고, 짧은 일기와 함께 뽑은 괘를 만다라 원에 새겨 자기 자리를 점검하는 정적 웹앱입니다. 단일 `index.html` 파일이며, 백엔드는 Supabase Postgres를 사용합니다.

## 업데이트 내역

| 버전 | 주요 변경 |
|------|-----------|
| **v2.1** | 노자(도덕경)·장자(소요유) 항목 추가. 각 괘마다 두 성인의 결로 오늘을 풀이 |
| v2.0 | 비밀번호 로그인·회원가입, 개인별 일기 격리, 만다라 궤적 시각화, 관리자 대시보드, 레거시 일기 자동 이관 |
| v1.2 | 로그인 게이트·방문자 카운터·흔들기 일진괘·날짜 시드 정세 |
| v1.1 | Supabase 연동, 일기 클라우드 저장 |
| v1.0 | 64괘 격자·정감록·예수제자·주식·불교 6대 풀이 |

## v2 구성

- **프론트엔드** — `index.html` 한 파일 (HTML + 인라인 CSS + 바닐라 JS + SVG 만다라)
- **인증** — `crypto.subtle.digest('SHA-256')` 기반 salt+해시. 세션은 `localStorage`
- **Supabase JS** — `@supabase/supabase-js@2` CDN
- **DB** — Supabase Postgres 테이블 `public.index_items_64` (v1 스키마 그대로 재활용)

## 데이터 모델

기존 `index_items_64` 테이블 하나에 두 종류의 레코드를 저장합니다. 스키마 변경 없이 v2 기능을 얹었습니다.

| 레코드 종류 | `id` 프리픽스 | `text`  | `hex` |
|-------------|---------------|---------|-------|
| 사용자 계정 | `usr:<닉네임>` | JSON `{kind:'user', nick, salt, passHash, isAdmin, createdAt}` | null |
| 일기       | `dia:<닉네임>:<YYYY-MM-DD>:<rand>` | JSON `{kind:'diary', owner, body, entryDate}` | 0~63 |
| (레거시)   | `d_...` (구버전) | 평문 일기 | 0~63 (있으면) |

관리자가 처음 로그인하면 남아 있는 `d_...` 레거시 레코드가 자동으로 관리자 소유 `dia:admin:...`로 이관됩니다.

### 사용자 격리

일반 사용자는 `SELECT * FROM index_items_64 WHERE id LIKE 'dia:<my-nick>:%'` 만 로드합니다. 관리자만 `usr:%` / `dia:%` 전체를 스캔합니다.

> anon 키가 여전히 모든 CRUD를 허용하므로 이 격리는 UI 수준의 편의성이며, 강한 보안 격리가 필요하면 Supabase RLS에서 `id LIKE 'dia:' || auth.jwt() ->> 'sub' || ':%'` 정책 도입을 권장합니다.

## 만다라 궤적

- 64분할 원 (매 8칸마다 굵은 방사선)
- 각 일기의 반경은 시간순(가장 오래된 것 = 안쪽, 최신 = 바깥쪽) 정규화
- 오늘 저장된 점은 오렌지, 과거는 금색
- 점 사이는 점선(연한 옥색)으로 궤적선 연결
- 통계: 기록 점 수, 기록 일수, 뽑은 괘의 다양성(N/64), 시작일/최근일

## 관리자 기능

닉네임 `admin`으로 가입한 계정은 자동으로 `isAdmin: true`가 됩니다.

- 전체 사용자 목록 + 각자의 일기 편수 표시
- 사용자 카드 클릭 시 해당 사용자의 일기만 필터링
- 사용자 삭제(계정 + 일기 일괄)
- 전체 JSON 백업 다운로드

## 로컬 실행

```bash
python -m http.server 8790
# http://127.0.0.1:8790/index.html
```

## 설정값

`index.html` 인라인 스크립트에서 Supabase 상수를 수정하면 다른 프로젝트로 연결 가능합니다.

```js
const SUPABASE_URL      = 'https://<project-ref>.supabase.co';
const SUPABASE_ANON_KEY = 'sb_publishable_...';
const TABLE_NAME        = 'index_items_64';
```

# 66index-creative-app

> 오늘의 64괘 — 정감록·예수제자의 길

주역 64괘를 토대로 그날의 흐름을 읽고, 짧은 일기로 자기 자리를 점검하는 정적 웹앱입니다. 단일 `index.html` 파일이며, 백엔드는 Supabase Postgres를 사용합니다.

## 구성

- **프론트엔드** — `index.html` 한 파일 (HTML + 인라인 CSS + 바닐라 JS)
- **Supabase JS** — `@supabase/supabase-js@2` CDN (`jsdelivr`)
- **DB** — Supabase Postgres 테이블 `public.index_items_64`

## Supabase 연동

일기 데이터는 모두 `public.index_items_64` 테이블에 저장됩니다. 로컬 스토리지는 미저장 초안(`iching_diary_draft_v1`)과 편집 중 ID(`iching_diary_current_v1`), 그리고 1회성 마이그레이션 표시(`iching_diary_migrated_to_supabase_v1`) 용도로만 사용됩니다.

### 테이블 스키마

```sql
create table public.index_items_64 (
  id          text primary key,            -- 클라이언트 생성 ID (예: d_<timestamp>_<rand>)
  text        text not null,
  hex         smallint check (hex is null or (hex >= 0 and hex < 64)),
  created_at  timestamptz not null default now(),
  updated_at  timestamptz not null default now()
);

create index index_items_64_updated_at_idx
  on public.index_items_64 (updated_at desc);

alter table public.index_items_64 enable row level security;
```

### RLS 정책

정적 페이지에서 anon 키로 직접 접근하므로, `anon` / `authenticated` 역할에 대해 SELECT / INSERT / UPDATE / DELETE 를 모두 허용하는 단순 정책을 사용합니다. 공개 데모용 설정이며, 멀티 유저 운영시에는 `user_id` 컬럼 + `auth.uid()` 기반 정책으로 강화하는 것을 권장합니다.

### 클라이언트 측 호출

`index.html` 내부에서 사용하는 주요 함수:

| 함수 | 동작 |
|------|------|
| `fetchDiariesFromCloud()` | `updated_at desc` 정렬로 전체 일기 로드 |
| `upsertDiaryToCloud(row)` | 신규/수정 일기 1건 upsert |
| `deleteDiaryFromCloud(id)` | 일기 1건 삭제 |
| `deleteAllDiariesFromCloud()` | "전체 삭제" 버튼용 |
| `migrateLocalToCloudIfNeeded()` | 첫 로드 시 기존 `iching_diary_v1` 로컬 항목을 클라우드로 일회 이전 |

UI 응답성을 위해 `diariesCache` 인메모리 배열을 함께 두고, 변경 즉시 캐시·DOM을 갱신한 뒤 비동기로 Supabase에 반영합니다.

## 로컬 실행

`file://` 프로토콜은 일부 브라우저 정책상 제한이 있으므로, 간단한 정적 서버를 띄워 여는 것을 권장합니다.

```bash
# 저장소 루트에서
python -m http.server 8765
# 브라우저에서 http://127.0.0.1:8765/index.html
```

별도의 빌드 단계나 패키지 설치는 없습니다.

## 설정값

`index.html` 인라인 스크립트의 다음 상수를 수정해 다른 Supabase 프로젝트로 연결할 수 있습니다.

```js
const SUPABASE_URL      = 'https://<project-ref>.supabase.co';
const SUPABASE_ANON_KEY = 'sb_publishable_...';
const TABLE_NAME        = 'index_items_64';
```

> anon 키는 클라이언트에 노출되는 것이 정상입니다. 실제 보안은 RLS 정책으로 강제하세요.

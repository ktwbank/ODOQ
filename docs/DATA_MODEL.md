# ODOQ 데이터 모델 & API 초안 (v0.1)

전제: Postgres(Supabase) + RLS. 모든 시각은 `timestamptz`(UTC 저장), 날짜 판정은 `entry_date`(date) 컬럼으로.

## 테이블

### profiles
| 컬럼 | 타입 | 비고 |
|---|---|---|
| id | uuid PK | auth.users.id 참조 |
| handle | text unique | 소문자 영숫자+언더스코어, 3~20자 |
| display_name | text | |
| bio | text | 최대 80자 |
| avatar_url | text | null 허용 |
| timezone | text | IANA (예: Asia/Seoul) |
| created_at | timestamptz | |

### entries
| 컬럼 | 타입 | 비고 |
|---|---|---|
| id | uuid PK | |
| author_id | uuid FK → profiles | |
| body | text | 1~140자 (CHECK) |
| visibility | text | 'public' \| 'private' |
| entry_date | date | 작성자 로컬 날짜 |
| like_count | int | 비정규화 카운터 (트리거 갱신) |
| is_hidden | bool | 신고 누적/운영 조치 |
| created_at / updated_at | timestamptz | |

**UNIQUE (author_id, entry_date)** ← "하루 1개" 제약의 실제 강제 지점.

### follows
| 컬럼 | 타입 |
|---|---|
| follower_id | uuid FK |
| followee_id | uuid FK |
| created_at | timestamptz |

PK (follower_id, followee_id), CHECK follower_id <> followee_id.

### likes
PK (entry_id, user_id) + created_at.

### blocks
PK (blocker_id, blocked_id).

### reports
| 컬럼 | 타입 |
|---|---|
| id | uuid PK |
| reporter_id | uuid |
| entry_id | uuid |
| reason | text ('spam','abuse','sexual','other') |
| status | text ('open','resolved','dismissed') |
| created_at | timestamptz |

### devices (푸시)
user_id, expo_push_token, platform, reminder_hour(int 0-23), updated_at.

## RLS 요지

| 테이블 | 정책 |
|---|---|
| entries SELECT | `visibility='public' AND is_hidden=false AND 차단관계 아님` OR `author_id = auth.uid()` |
| entries INSERT | `author_id = auth.uid()` (유니크 제약이 1일 1개 보장) |
| entries UPDATE | `author_id = auth.uid() AND entry_date = current_date_in_author_tz` |
| entries DELETE | `author_id = auth.uid()` |
| follows INSERT/DELETE | `follower_id = auth.uid()` |
| likes | 본인 행만 쓰기, 읽기는 공개 글 한정 |

## API (Supabase 직접 쿼리 + 일부 RPC)

| 동작 | 방식 |
|---|---|
| 오늘 글 조회 | `select * from entries where author_id=$me and entry_date=$today` |
| 글 작성 | `insert into entries ...` → 23505 중복 오류 = 이미 작성함 |
| 전체 피드 | `select ... where visibility='public' and is_hidden=false and created_at > now()-interval '24 hours' order by created_at desc limit 20` (cursor: created_at, id) |
| 팔로잉 피드 | RPC `feed_following(cursor, limit)` — follows 조인, 차단 제외 |
| 팔로우 | `insert into follows` / `delete from follows` |
| 공감 | `insert into likes` + 트리거로 `entries.like_count` 갱신 |
| 프로필 조회 | profiles + 공개 entries 페이지네이션 |
| 신고 | `insert into reports`; 트리거로 open 신고 3건이면 `is_hidden=true` |

## 인덱스

- `entries (created_at desc)` — 전체 피드
- `entries (author_id, entry_date desc)` — 내 기록
- `follows (follower_id)` — 팔로잉 피드
- `likes (user_id)` — 내 공감 여부 표시

## 성능 주의 (추정)

팔로잉 피드는 `follows ⋈ entries` 조인으로 시작해도 팔로잉 수천 명·전체 글 수백만 건 전까지 문제 없음(추정). 그 이후엔 fan-out 캐시 테이블(`feed_items`) 도입을 검토.

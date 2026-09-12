# ODOQ 데이터 모델 & API 초안 (v0.2)

전제: Postgres + RLS. 시각은 `timestamptz`(UTC), 날짜 판정은 `entry_date`(date, 작성자 로컬 기준).
v0.1 → v0.2: **1일 1개 유니크 제약 제거**, 대표 구절(`is_primary`) 도입, 500자, 토픽/추천, 동의 이력, 모더레이션 테이블 추가.

---

## 테이블

### profiles
| 컬럼 | 타입 | 비고 |
|---|---|---|
| id | uuid PK | auth 사용자 id |
| handle | text unique | 소문자 영숫자+`_`, 3~20자 |
| display_name | text | |
| bio | text | 80자 |
| avatar_url | text | |
| timezone | text | IANA (Asia/Seoul) |
| is_seed_creator | bool | 시드 크리에이터 배지·추천 가중 |
| is_recommendable | bool | 추천 화이트리스트 (초기 수동 큐레이션) |
| status | text | 'active' / 'suspended' / 'deleted' |
| created_at | timestamptz | |

### entries
| 컬럼 | 타입 | 비고 |
|---|---|---|
| id | uuid PK | |
| author_id | uuid FK → profiles | |
| body | text | **1~500자 CHECK** |
| visibility | text | 'public' / 'private' |
| entry_date | date | 작성자 로컬 날짜 |
| is_primary | bool | 그날의 **대표 구절** |
| like_count | int | 비정규화 카운터(트리거) |
| is_hidden | bool | 신고 누적/운영 임시조치 |
| hidden_reason | text | 'report_threshold' / 'moderation' / 'copyright' |
| created_at / updated_at | timestamptz | |

제약:
```sql
-- 하루 여러 개 허용. 단 대표 구절은 날짜당 1개.
CREATE UNIQUE INDEX entries_one_primary_per_day
  ON entries (author_id, entry_date) WHERE is_primary;

ALTER TABLE entries ADD CONSTRAINT body_len
  CHECK (char_length(body) BETWEEN 1 AND 500);

-- 수정 가능 시간: created_at + 24h (트리거로 강제)
```

### topics / user_topics / entry_topics
| 테이블 | 컬럼 |
|---|---|
| topics | id, slug, label, sort_order |
| user_topics | user_id, topic_id (PK 복합) — 온보딩 관심사 |
| entry_topics | entry_id, topic_id — v1은 내부 큐레이션용(유저 노출 여부 미정) |

### follows
follower_id, followee_id (PK 복합), created_at. `CHECK follower_id <> followee_id`.

### likes
entry_id, user_id (PK 복합), created_at.

### blocks
blocker_id, blocked_id (PK 복합), created_at. **양방향 비노출 적용.**

### mutes
muter_id, muted_id — 차단보다 약한 "이 사용자 글 숨기기".

### reports
| 컬럼 | 타입 |
|---|---|
| id | uuid PK |
| reporter_id | uuid |
| target_type | text ('entry' / 'user') |
| target_id | uuid |
| category | text ('spam','hate','sexual','defamation','copyright','selfharm','other') |
| detail | text |
| status | text ('open','under_review','actioned','dismissed') |
| created_at / resolved_at | timestamptz |

### moderation_actions (운영 이력 — 법적 입증용)
id, actor(운영자 id 또는 'system'), target_type, target_id, action('hide','unhide','suspend','delete','warn'), reason, related_report_id, created_at.

### appeals (이의신청)
id, user_id, moderation_action_id, body, status('open','accepted','rejected'), created_at.

### consents (동의 이력 — 입증 책임 대응)
| 컬럼 | 타입 |
|---|---|
| id | uuid PK |
| user_id | uuid |
| doc_type | text ('tos','privacy','marketing','night_marketing','age14') |
| doc_version | text |
| granted | bool |
| ip | inet |
| created_at | timestamptz |

### devices
user_id, expo_push_token, platform, reminder_enabled, reminder_hour(0-23), like_noti, follow_noti, marketing_noti, updated_at.

---

## RLS 요지

| 테이블 | 정책 |
|---|---|
| entries SELECT | (`visibility='public' AND is_hidden=false AND status='active' AND 차단관계 없음`) OR `author_id = auth.uid()` |
| entries INSERT | `author_id = auth.uid()` (횟수 제한 없음) |
| entries UPDATE | `author_id = auth.uid() AND now() < created_at + interval '24 hours'` |
| entries DELETE | `author_id = auth.uid()` |
| follows | `follower_id = auth.uid()`만 쓰기 |
| likes | 본인 행만 쓰기, 읽기는 노출 가능한 글 한정 |
| reports | INSERT는 본인, SELECT는 운영자만 |
| consents | INSERT 본인, UPDATE 불가(append-only) |
| moderation_actions | 운영자 role만 |

> RLS 테스트는 출시 차단 항목이다. 최소 케이스: ①비공개 글이 타인 조회에 0건 ②차단 관계에서 양방향 0건 ③is_hidden 글이 작성자 외 0건 ④24h 경과 글 UPDATE 거부.

---

## 주요 쿼리 / RPC

| 동작 | 방식 |
|---|---|
| 오늘 내 구절 | `where author_id=$me and entry_date=$today order by created_at` |
| 작성 | `insert into entries(...)` → 그날 첫 글이면 `is_primary=true` |
| 대표 지정 | RPC `set_primary(entry_id)` — 같은 날 기존 대표 해제 후 지정(트랜잭션) |
| 대표 삭제 시 승계 | 트리거: 같은 `(author_id, entry_date)` 중 가장 오래된 글에 `is_primary` 이전 |
| 팔로잉 피드 | RPC `feed_following(cursor, limit)` — follows 조인, 차단·뮤트·hidden 제외, `(created_at, id)` 커서 |
| 추천 피드 | RPC `feed_recommended(cursor, limit)` — 아래 §추천 |
| 추천 유저 | RPC `suggest_users(limit)` — 온보딩·빈 피드용 |
| 공감 | `insert into likes` + 트리거로 `like_count` 갱신 |
| 신고 | `insert into reports`; 트리거로 동일 대상 open 3건 → `is_hidden=true` + moderation_actions 기록 |
| 계정 삭제 | RPC `delete_account()` — entries/likes/follows/devices 하드 삭제, profiles 파기, 신고 이력은 가명화 |

### 추천 로직 (v1 휴리스틱, SQL 표현)

```sql
-- 추천 유저
SELECT p.id
FROM profiles p
JOIN user_topics ut ON ut.user_id = p.id
WHERE p.is_recommendable
  AND p.status = 'active'
  AND ut.topic_id = ANY($my_topics)
  AND NOT EXISTS (SELECT 1 FROM follows f WHERE f.follower_id=$me AND f.followee_id=p.id)
  AND NOT EXISTS (SELECT 1 FROM blocks b WHERE (b.blocker_id=$me AND b.blocked_id=p.id)
                                            OR (b.blocker_id=p.id AND b.blocked_id=$me))
  AND (SELECT count(*) FROM entries e
       WHERE e.author_id=p.id AND e.created_at > now() - interval '14 days') >= 3
ORDER BY (recent_like_avg(p.id) * 0.7 + newcomer_boost(p.id) * 0.3) DESC
LIMIT $n;
```

`newcomer_boost`: 가입 30일 이내 + 누적 공감 5 이하인 작성자를 소량 끌어올린다. **신규의 첫 공감 경험이 리텐션의 시작점**이기 때문(SEEDING.md 참조).

---

## 인덱스

| 인덱스 | 용도 |
|---|---|
| `entries (created_at DESC) WHERE visibility='public' AND NOT is_hidden` | 추천 피드 |
| `entries (author_id, entry_date DESC)` | 내 기록 |
| `entries (author_id, entry_date) WHERE is_primary` (unique) | 대표 구절 |
| `follows (follower_id)` / `follows (followee_id)` | 피드 / 팔로워 수 |
| `likes (user_id)` | 내 공감 여부 |
| `reports (status, created_at)` | 운영 큐 |

## 성능 주의 (추정)

`follows ⋈ entries` 조인 방식의 팔로잉 피드는 팔로잉 수천 명 · 전체 글 수백만 건 수준까지는 문제되지 않을 것으로 **추정**한다(실측 필요). 그 이후 fan-out 캐시 테이블(`feed_items`)을 도입한다. 500자 본문은 피드에서 **왼쪽 200자만 잘라 전송**해 페이로드를 줄인다(상세 진입 시 전문 조회).

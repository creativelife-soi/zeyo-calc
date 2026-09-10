# 하루 목표 칼로리 계산기 — 배포 & 보안 노트

정적 파일 하나(`index.html`)로 끝납니다. 빌드 없음, 의존성 없음.
집계 대시보드는 사이트에서 제거했고, Supabase SQL Editor에서 봅니다.

---

## 1. 띄워보기

```bash
python3 -m http.server 8000   # http://localhost:8000
```

`CONFIG`가 비어 있으면 이벤트는 브라우저 콘솔에만 찍힙니다(`[track] …`). 서버로 나가지 않습니다.

## 2. 배포

파일이 `index.html` 하나뿐이라 정적 호스팅이면 어디든 됩니다. 순서는 **Supabase 먼저, 배포는 나중**입니다.
키를 넣지 않은 채 배포하면 그동안의 방문 기록이 남지 않습니다.

### 권장: GitHub + Vercel (푸시하면 자동 재배포)

```bash
cd <이 폴더>
git init
git add index.html README.md
git commit -m "반려동물 하루 목표 칼로리 계산기"
gh repo create zeyo-calc --public --source=. --push
# gh CLI가 없으면 GitHub에서 빈 저장소를 만든 뒤
# git remote add origin https://github.com/<계정>/zeyo-calc.git && git push -u origin main
```

vercel.com → Add New → Project → Import Git Repository → 저장소 선택 →
Framework Preset은 **Other**, 나머지 기본값 → Deploy.

이후에는 `git push`만 하면 자동으로 다시 배포됩니다. 문구를 고칠 일이 많으니 이 방식이 편합니다.

앱 저장소(`creativelife-soi/ZEYO`)와 **분리된 새 저장소**를 쓰세요. 이건 검증용 랜딩페이지이고,
앱 코드와 배포 주기가 다릅니다.

### 더 빠른 방법: Netlify Drop

app.netlify.com/drop 에 폴더를 끌어다 놓으면 끝입니다. 계정 없이도 주소가 나옵니다.
대신 수정할 때마다 다시 끌어다 놔야 하고 주소가 무작위입니다. 하루 이틀 테스트용으로만.

### CLI 한 줄

```bash
npx vercel --prod
```

### 커밋해도 되는 것 / 안 되는 것

- `index.html`의 **anon 키는 커밋해도 됩니다.** 원래 공개용입니다.
- `service_role` 키는 어디에도 넣지 마세요. 이 프로젝트에는 아예 필요 없습니다.
- 배포 직후 5-5 체크리스트의 읽기 차단 테스트를 꼭 한 번 돌려보세요.

---

## 3. Supabase 세팅

### 3-0. API 설정 토글 (Settings → API)

| 토글 | 설정 | 이유 |
|---|---|---|
| Enable Data API | **켜기** | 사이트가 `/rest/v1/events`로 기록을 보냅니다. 끄면 이 경로 자체가 사라져 수집이 전부 실패합니다. |
| Automatically expose new tables | **끄기** | 켜면 `public`에 새로 만드는 테이블·뷰에 `anon` 권한이 자동으로 붙습니다. 5-2에서 설명한 뷰 우회 사고의 원인입니다. 아래 SQL이 필요한 권한을 직접 부여하므로 꺼도 정상 동작합니다. |
| Enable automatic RLS | **켜기** | `public`의 새 테이블에 RLS를 자동으로 켜 줍니다. 정책을 깜빡해도 "전부 차단"으로 실패하니 안전한 쪽입니다. 아래 SQL도 명시적으로 켜므로 중복돼도 무해합니다. |

**Exposed schemas** 에는 `public`(과 기본값 `graphql_public`)만 두고 `analytics`는 넣지 마세요.

토글 순서는 상관없습니다. 아래 SQL의 `revoke all` → `grant insert (컬럼)` 조합이
이미 붙어 있던 기본 권한까지 정리하기 때문에, 테이블을 먼저 만들었어도 결과는 같습니다.

### 3-1. 스키마 (SQL Editor에 그대로 붙여넣기)

```sql
-- ─────────────────────────────────────────────
-- 이벤트 (익명 행동 로그)
-- ─────────────────────────────────────────────
create table public.events (
  id          bigint generated always as identity primary key,
  created_at  timestamptz not null default now(),
  site        text not null,
  visitor_id  text,
  session_id  text,
  name        text not null,
  props       jsonb not null default '{}'::jsonb,
  constraint events_site_ck  check (site = 'zeyo-calc'),
  constraint events_name_ck  check (char_length(name) between 1 and 40),
  constraint events_vid_ck   check (visitor_id is null or char_length(visitor_id) <= 32),
  constraint events_sid_ck   check (session_id is null or char_length(session_id) <= 32),
  constraint events_props_ck check (pg_column_size(props) <= 2048)
);

create index events_created_idx on public.events (created_at desc);
create index events_name_idx    on public.events (name);
create index events_sess_idx    on public.events (session_id);

alter table public.events enable row level security;

revoke all on public.events from anon, authenticated;
grant insert (site, visitor_id, session_id, name, props) on public.events to anon;

create policy "anon insert only" on public.events
  for insert to anon with check (true);

-- ─────────────────────────────────────────────
-- 이메일 신청 (개인정보 — 분리 보관)
-- ─────────────────────────────────────────────
create table public.signups (
  id          bigint generated always as identity primary key,
  created_at  timestamptz not null default now(),
  email       text not null unique,
  source      text not null,
  props       jsonb not null default '{}'::jsonb,
  constraint signups_email_ck  check (email ~ '^[^@[:space:]]+@[^@[:space:]]+\.[^@[:space:]]+$'
                                      and char_length(email) <= 120
                                      and email = lower(email)),
  constraint signups_source_ck check (source = 'zeyo-calc'),
  constraint signups_props_ck  check (pg_column_size(props) <= 512)
);

alter table public.signups enable row level security;

revoke all on public.signups from anon, authenticated;
grant insert (email, source, props) on public.signups to anon;

create policy "anon insert only" on public.signups
  for insert to anon with check (true);
```

### 3-2. 설정 채우기

`index.html` 상단:

```js
SUPABASE_URL:      'https://xxxx.supabase.co',
SUPABASE_ANON_KEY: 'eyJhbGci...',   // anon public key
```

`service_role` 키는 어디에도 넣지 않습니다. 이제 필요 없습니다.

---

## 4. 대시보드 = Supabase 안에서

집계 뷰는 **`public`이 아니라 `analytics` 스키마에** 만듭니다. 이유는 5-2에 있습니다.

```sql
create schema if not exists analytics;
revoke all on schema analytics from anon, authenticated;

-- 일자별 (한국시간 기준)
create or replace view analytics.daily as
select (created_at at time zone 'Asia/Seoul')::date        as "날짜",
       count(distinct session_id)                          as "방문",
       count(distinct visitor_id)                          as "방문자",
       count(distinct session_id) filter (where name='step1_done')      as "계산시작",
       count(distinct session_id) filter (where name='step3_view')      as "결과확인",
       count(distinct session_id) filter (where name='click_cta_notify') as "알림신청"
from public.events
group by 1 order by 1 desc;

-- 퍼널 (전체 누적)
create or replace view analytics.funnel as
select count(distinct session_id)                                        as "방문",
       count(distinct session_id) filter (where name='step1_done')       as "1단계완료",
       count(distinct session_id) filter (where name='step3_view')       as "결과확인",
       count(distinct session_id) filter (where name='click_cta_notify')  as "알림신청",
       round(100.0 * count(distinct session_id) filter (where name='step3_view')
             / nullif(count(distinct session_id),0), 1)                   as "결과도달률",
       round(100.0 * count(distinct session_id) filter (where name='click_cta_notify')
             / nullif(count(distinct session_id) filter (where name='step3_view'),0), 1)
                                                                          as "신청전환율"
from public.events;

-- 버튼 클릭
create or replace view analytics.clicks as
select name                                                as "이벤트",
       count(*)                                            as "횟수",
       count(distinct session_id)                          as "세션수"
from public.events
where name like 'click_%'
group by 1 order by 2 desc;

-- 입력값 분포
create or replace view analytics.inputs as
select props->>'species'                                   as "종",
       props->>'mode'                                      as "모드",
       count(*)                                            as "건수",
       round(avg((props->>'loss_pct')::numeric), 1)        as "평균감량목표_%",
       round(avg((props->>'cur_kg')::numeric), 2)          as "평균현재체중_kg",
       count(*) filter (where (props->>'phased')::boolean) as "구간분할_건수"
from public.events
where name = 'step1_done'
group by 1,2 order by 3 desc;
```

**Settings → API → Exposed schemas** 에 `analytics`가 **없는지** 확인하세요. `public`만 있으면 됩니다.

### 매일 보는 쿼리

```sql
select * from analytics.funnel;
select * from analytics.daily limit 14;
select * from analytics.clicks;
select * from analytics.inputs;
select created_at, email, props from public.signups order by created_at desc;
```

---

## 5. 보안 점검 결과

### 5-1. anon 키가 소스에 노출되는 건 문제가 아닙니다

anon 키는 원래 공개용입니다. 실제 방어선은 RLS이고, 위 스키마는 이렇게 잠급니다.

| 항목 | 조치 |
|---|---|
| 읽기 | `select` 정책 없음 + `revoke all` → anon은 조회 불가 |
| 쓰기 컬럼 | 컬럼 단위 `grant insert` → 지정한 컬럼 외 삽입 불가 |
| 시각 위조 | `created_at`을 클라이언트가 안 보냄, 서버 `now()` 고정 |
| id 위조 | `generated always as identity` → 클라이언트가 지정 불가 |
| 페이로드 비대 | `pg_column_size(props)` CHECK로 상한 |
| 응답 유출 | 요청에 `Prefer: return=minimal` → INSERT 결과도 안 돌려받음 |

### 5-2. ⚠️ 가장 중요한 함정 — 뷰는 RLS를 우회합니다

Supabase는 `public` 스키마의 새 테이블·뷰에 `anon` SELECT 권한을 기본으로 부여합니다.
테이블은 RLS가 막아주지만, **뷰는 기본적으로 소유자(postgres) 권한으로 실행돼 RLS를 통과합니다.**
집계 뷰를 `public`에 만들면 anon 키만으로 `/rest/v1/daily` 같은 경로에서 전부 읽힙니다.

그래서 위에서 뷰를 `analytics` 스키마에 만들고 API 노출에서 제외했습니다.
`public`에 만들어야 한다면 반드시 둘 중 하나를 해주세요.

```sql
revoke all on analytics.daily from anon, authenticated;
-- 또는 (PG15+)
alter view public.daily set (security_invoker = on);
```

### 5-3. 남는 위험 — 스팸 삽입

anon 키가 공개된 이상, 누군가 스크립트로 가짜 이벤트나 가짜 이메일을 밀어 넣을 수 있습니다.
무료 티어에서 완전 차단은 불가능하고, 실무적으로는 이렇게 다룹니다.

- **탐지**: 하루 방문이 갑자기 튀면 `visitor_id`별 건수를 확인
  ```sql
  select visitor_id, count(*) from public.events
  where created_at > now() - interval '1 day'
  group by 1 order by 2 desc limit 20;
  ```
- **정리**: 특정 `visitor_id` 삭제 후 수치 재확인
- **대응**: 심하면 Supabase에서 anon 키 회전(rotate) → `index.html` 갱신
- **예방**: 이메일 폼에 Cloudflare Turnstile 같은 캡차를 붙이면 대부분 막힙니다 (지금은 미적용)

수요 검증이 목적이므로 초기에는 탐지·정리 수준으로 충분합니다. 다만 **알림 신청 수를 근거로 판단하기 전에 반드시 위 쿼리로 한 번 걸러보세요.**

### 5-4. 개인정보 (이메일)

이메일은 1회 발송용이 아니라 **지속적인 소식 발송용**으로 수집합니다. 그만큼 관리 의무가 따릅니다.

- 이메일은 `events`가 아닌 **`signups` 테이블에 분리** 저장합니다. `props`에는 절대 넣지 않습니다.
- 사이트에 동의 체크박스와 수집 항목·이용 목적·보유 기간·동의 거부권·수신 거부 방법·문의처를 고지했습니다.
  개인정보보호법 제15조가 요구하는 항목입니다.
- 앱 소식 메일은 영리목적 광고성 정보에 해당할 수 있습니다. **정보통신망법 제50조**에 따라
  발송할 때마다 다음을 지키세요.
  - 제목 맨 앞에 `(광고)` 표기
  - 본문에 발신자 명칭·연락처, 그리고 수신 거부 방법을 명확히 표시
  - 오후 9시~오전 8시 사이에는 별도 야간 수신 동의 없이 발송 금지
- **2년마다 수신 동의 여부를 다시 확인**해야 합니다(정보통신망법 시행령 제62조의3).
  최초 동의일 기준이므로 `created_at`으로 대상을 뽑으세요.
  ```sql
  select email, created_at from public.signups
  where created_at < now() - interval '2 years' order by created_at;
  ```
- 수신 거부나 삭제 요청이 오면 해당 행을 지웁니다.
  ```sql
  delete from public.signups where email = 'name@example.com';
  ```
- 서비스를 접거나 더 이상 발송하지 않기로 하면 전체를 파기하세요. 보유 기간 고지에 그렇게 적혀 있습니다.
  ```sql
  truncate public.signups;
  ```
- IP는 앱에서 수집하지 않지만, Supabase 인프라 로그에는 남습니다. 프로젝트 로그 보존 기간을 확인해 두세요.

### 5-5. 체크리스트

- [ ] 두 테이블 모두 `rls enabled` (Table Editor에 "Unrestricted" 배지가 없어야 함)
- [ ] `analytics`가 Exposed schemas에 없음
- [ ] Automatically expose new tables = 꺼짐 / Enable automatic RLS = 켜짐
- [ ] `service_role` 키가 저장소·프론트엔드 어디에도 없음
- [ ] 배포 후 브라우저 콘솔에서 읽기가 막히는지 실제 확인:
  ```js
  fetch(URL+'/rest/v1/events?select=*', {headers:{apikey:ANON, Authorization:'Bearer '+ANON}})
    .then(r=>r.json()).then(console.log)   // [] 또는 권한 오류가 나와야 정상
  ```

---

## 6. 수집 이벤트

| 이름 | 시점 | props |
|---|---|---|
| `visit` | 페이지 진입 | `ref`, `w` |
| `click_calc` | 1단계 계산 버튼 | – |
| `step1_done` | 1단계 완료 | `species`, `mode`, `cur_kg`, `tgt_kg`, `loss_pct`, `neutered`, `phased` |
| `step2_done` | 2단계 통과 | `species`, `rer` |
| `step3_view` | 결과 도달 | `species`, `kcal`, `factor`, `phased` |
| `click_factor_change` | 계수 변경 | `factor` |
| `click_cta_notify` | 알림 신청 성공 | `species`, `mode` |
| `cta_error` | 신청 실패 | – |
| `click_restart` | 다시 계산 | – |

`visitor_id`는 브라우저에 저장되는 익명 난수이고, 개인 식별 정보가 아닙니다.

---

## 7. 계산식과 근거

### 기본 계산

- RER = 70 × (목표체중kg)^0.75 · 하루 목표 = RER × 계수 — AAHA 2021 Guidelines, Box 1
- 감량 계수: 고양이 0.8 / 강아지 1.0
- **열량은 처음부터 끝까지 최종 목표(이상) 체중 기준으로 계산합니다.** 중간 목표 체중으로 다시 계산하지 않습니다.

### 왜 "중간 목표마다 열량 재계산"을 버렸나

초판에서는 감량 폭이 크면 1차·2차 목표로 쪼개고 각 구간의 목표 체중으로 열량을 다시 계산했습니다.
결과적으로 초기 급여량이 최종 목표 기준보다 10~15% 높아졌는데, 이는 근거와 반대 방향이었습니다.

1. German(2016)의 완주 코호트에서 감량 속도는 초기 주당 1.2%에서 672일차 0.1%까지 떨어졌고,
   그동안 **열량은 계속 줄이고 있었는데도** 그랬습니다. 대사 적응은 열량을 더 내려야 상쇄됩니다.
   시작을 느슨하게 잡으면 정체가 더 빨리, 더 깊게 옵니다.
2. German(2011): 감량 후 유지 열량은 감량기 열량보다 약 10% 높은 데 그쳤습니다. 대사 효율이 올라갑니다.
3. 실제 임상 급여량은 가이드라인 시작값보다 낮습니다.
   개는 목표 체중 MER의 55~65% 수준, 고양이 클리닉 실측 평균은 53 kcal/이상체중kg^0.67
   (4.5kg 고양이 기준 약 145 kcal — AAHA 시작값 173 kcal의 84%).

### 대신 넣은 것

| 요소 | 값 | 근거 |
|---|---|---|
| 목표 감량 속도 | 고양이 주당 ~1%, 강아지 주당 ~1.5% | AAHA / Tufts / Purina |
| 재측정 주기 | 2주 | German 2016 (클리닉 표준 2~4주) |
| 정체 시 조정 | 4주간 미달이면 −10%, 반복 가능 | 임상 관행 (APOP 단계 계산기 동일 구조) |
| 과속 시 조정 | 상한 초과면 +10% | 동일 |
| 하한선 | 목표 체중 RER의 60% | AAHA 2014가 단백질 적정성 표를 RER 80%·60% 기준으로 제시 (APOP는 70% 사용) |
| 간식 | 총열량의 10% 이내 | APOP |
| 재평가 지점 | 시작 체중의 5%씩 (열량은 불변) | 6% 이상 감량에서 이미 관절·삶의 질 개선 관측 (Marshall 2010) |

### 부분 감량 (감량 폭 20% 초과 시 제안)

목표를 이상 체중보다 높게 잡고 **거기서 종료**하는 방식. 이상 체중 대비 +15% 지점을 제안합니다
(German 2023의 부분 감량군 중앙값 +18%).

핵심은 **하루 열량은 그대로 두고 결승선만 앞당긴다**는 것입니다. 해당 연구에서 완전 감량군과
부분 감량군의 실제 섭취량은 차이가 없었고(양쪽 53 kcal/이상체중kg^0.67, p=0.589),
그럼에도 부분 감량군이 주당 감량 속도가 빨랐으며(0.81% vs 0.61%, p=0.028)
방문 횟수가 적었고(7회 vs 11회) 제지방량이 유의하게 줄지 않았습니다.
대상은 고령(>9세) 또는 심한 비만(이상 체중 대비 +40% 초과)이었습니다.

### 출처

- AAHA. 2021 Nutrition and Weight Management Guidelines, Box 1.
  https://www.aaha.org/resources/2021-aaha-nutrition-and-weight-management-guidelines/weight-reduction-in-the-obese-pet/
- German AJ. Outcomes of weight management in obese pet dogs: what can we do better? *Proc Nutr Soc* 2016.
  https://livrepository.liverpool.ac.uk/3001233/
- German AJ, et al. Low-maintenance energy requirements of obese dogs after weight loss. *Br J Nutr* 2011;106:S93-6.
  https://pubmed.ncbi.nlm.nih.gov/22005443/
- German AJ, Woods-Lee GRT, Biourge V, Flanagan J. Partial weight reduction protocols in cats lead to better
  weight outcomes, compared with complete protocols, in cats with obesity. *Front Vet Sci* 2023;10:1211543.
  https://pmc.ncbi.nlm.nih.gov/articles/PMC10318927/
- German AJ, et al. Assessing the adequacy of essential nutrient intake in obese dogs undergoing energy
  restriction for weight loss. *BMC Vet Res* 2015;11:253.
  https://bmcvetres.biomedcentral.com/articles/10.1186/s12917-015-0570-y
- APOP. Step Weight Loss Calculator (veterinary use).
  https://www.petobesityprevention.org/step-weight-loss-calculator
- Tufts Petfoodology. How fast is too fast for my pet to lose weight?
  https://vetnutrition.tufts.edu/2018/10/how-fast-is-too-fast-for-my-pet-to-lose-weight/

### 한계

이 계산기는 이상 체중을 사용자가 직접 입력받습니다. 실제 임상에서는 BCS(9점 척도) 또는 DEXA로
이상 체중을 먼저 추정합니다. BCS 7·8·9는 각각 현재 체중 ÷ 1.2 / 1.3 / 1.4로 근사합니다(German 2023 방법).
BCS 입력을 넣으면 목표 체중 추정의 정확도가 올라갑니다 — 다음 개선 후보.

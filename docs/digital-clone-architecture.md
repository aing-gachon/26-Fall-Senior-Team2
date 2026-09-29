# 디지털 클론 시스템 설계안

## 1. 문서 목적

이 문서는 300개 질문에 대한 사용자의 선택을 바탕으로 전역 프로필을 만들고, 임베딩 검색과 Jev 기반 선택지 그래프를 이용해 사용자의 판단을 예측하는 디지털 클론의 전체 설계를 정의한다.

시스템은 다음 다섯 영역으로 구성한다.

1. 공통 질문·선택지 전처리
2. 사용자 질문 제공 및 답변 수집
3. 질문 결과 기반 전역 프로필 생성
4. 디지털 클론과의 채팅
5. 사용자와 디지털 클론의 답변 유사도 평가

핵심 원칙은 다음과 같다.

- 전역 프로필은 모든 요청에 전달하는 안정적인 사용자 지침으로 사용한다.
- 현재 질문과 직접 관련된 원본 답변은 전역 프로필보다 우선한다.
- Jev는 글을 생성하는 용도가 아니라 관계 판정, 재정렬, 주장 검증에 사용한다.
- LLM은 프로필과 최종 답변처럼 자연어 종합이 필요한 곳에만 사용한다.
- 사용자가 선택하지 않은 선택지는 그 비교 상황에서 선택되지 않았다는 의미일 뿐, 항상 싫어한다는 의미가 아니다.
- 사용자별 300개 답변 관계를 다시 `300C2`로 평가하지 않는다. 공통 선택지 그래프에 사용자의 선택을 투영한다.

---

## 2. 전체 시스템 구성

```text
[맥북: 공통 전처리]
1,200개 선택지 임베딩
        ↓
선택지별 유사 노드 top 50
        ↓
Jev 관계 평가
        ↓
공통 선택지 그래프

[웹: 사용자 질문지]
300개 질문 제공
        ↓
문항별 답변 자동 저장
        ↓
질문지 완료

[Lambda: 프로필 생성]
사용자의 선택을 공통 그래프에 투영
        ↓
반복 경향·긴장·조건 추출
        ↓
LLM profile.json 생성
        ↓
Jev 주장 검증
        ↓
PROFILE.md 생성

[Lambda: 디지털 클론]
PROFILE.md + 벡터 검색 + 그래프 확장
        ↓
Jev 후보 재정렬
        ↓
최종 LLM 답변

[평가]
사용자와 클론이 같은 새 질문에 독립적으로 답변
        ↓
선택·이유·조건 비교
```

---

## 3. 기술 스택

| 영역 | 기술 | 역할 |
|---|---|---|
| 웹 | Next.js App Router, React, TypeScript | 질문지, 채팅, 평가 화면 |
| UI | Tailwind CSS | 반응형 인터페이스 |
| 인증 | Supabase Auth | 사용자 인증 |
| 데이터베이스 | Supabase PostgreSQL | 질문, 답변, 그래프, 상태 저장 |
| 벡터 검색 | Supabase pgvector | 선택지 및 사용자 문답 검색 |
| 비동기 처리 | AWS Lambda Python | 프로필 생성, 채팅, 평가 |
| 파일 저장 | Cloudflare R2 | 전처리 산출물과 프로필 파일 저장 |
| 공통 전처리 | macOS Python CLI | 임베딩 및 Jev 그래프 생성 |
| 관계 평가 | TypeSafe Jev | 관계 판정, 재정렬, 주장 검증 |
| 자연어 생성 | 구조화 출력을 지원하는 LLM | 프로필과 최종 응답 생성 |
| 웹 배포 | Vercel | Next.js 배포 |
| Python 관리 | uv | 의존성 및 실행 관리 |
| JavaScript 관리 | pnpm | 프론트엔드 패키지 관리 |

상시 실행하는 EC2, RDS, FastAPI 서버, Redis, Celery, 그래프 DB는 초기 버전에 사용하지 않는다.

---

# 영역 1. 공통 질문·선택지 전처리

## 1.1 목적

300개 질문과 질문별 4개 선택지를 총 1,200개의 선택지 노드로 변환한다. 각 노드를 임베딩하고, 임베딩으로 관련 후보를 좁힌 뒤 Jev가 관계를 판정한다.

이 작업은 질문지가 생성되거나 변경됐을 때 맥북에서 실행한다.

## 1.2 입력 데이터

```json
{
  "questionnaire_id": "default-v1",
  "questions": [
    {
      "id": "q001",
      "text": "새로운 업무를 고른다면?",
      "options": [
        { "id": "q001:o1", "text": "익숙한 업무" },
        { "id": "q001:o2", "text": "어려운 신규 프로젝트" },
        { "id": "q001:o3", "text": "보상이 높은 업무" },
        { "id": "q001:o4", "text": "부담이 적은 업무" }
      ]
    }
  ]
}
```

검증 사항:

- 질문이 정확히 300개인지 확인한다.
- 질문마다 선택지가 정확히 4개인지 확인한다.
- 질문과 선택지 ID가 중복되지 않는지 확인한다.
- 빈 문장이나 동일한 선택지가 없는지 확인한다.
- 질문지 버전과 각 노드의 콘텐츠 해시를 생성한다.

## 1.3 선택지 노드 문서 생성

선택지 문구만 사용하지 않고 질문과 전체 비교 선택지를 포함한다.

```text
질문: 새로운 업무를 고른다면?

비교 선택지:
- 익숙한 업무
- 어려운 신규 프로젝트
- 보상이 높은 업무
- 부담이 적은 업무

대상 선택:
어려운 신규 프로젝트
```

노드 구조:

```json
{
  "node_id": "q001:o2",
  "question_id": "q001",
  "question": "새로운 업무를 고른다면?",
  "all_options": [
    "익숙한 업무",
    "어려운 신규 프로젝트",
    "보상이 높은 업무",
    "부담이 적은 업무"
  ],
  "focal_option": "어려운 신규 프로젝트",
  "content_hash": "..."
}
```

## 1.4 전체 임베딩 생성

1,200개 노드 문서를 동일한 임베딩 모델로 변환한다.

```text
질문지 검증
  → 선택지 노드 문서 생성
  → 콘텐츠 해시 비교
  → 신규·변경 노드만 임베딩
  → pgvector 저장
```

저장 필드:

```text
option_nodes
- option_id
- question_id
- content
- content_hash
- embedding
- embedding_model
- questionnaire_version
```

## 1.5 top 50 관계 후보 생성

각 노드에 대해 다른 질문의 선택지 중 임베딩 유사도가 높은 50개를 선택한다.

```text
각 노드에서 전체 1,200개 검색
  → 같은 질문에 속한 선택지 제외
  → 코사인 유사도 top 50
  → (A, B)와 (B, A) 중복 제거
```

후보 수는 최대 약 60,000개다.

```text
1,200 × 50 = 60,000 directed candidates
중복 제거 후 약 30,000~60,000 unique pairs
```

초기에는 top 50으로 시작하고, 반대 관계가 충분히 검색되지 않는다면 top 100으로 확장한다.

## 1.6 Jev 관계 평가

각 후보 쌍에 대해 관련성과 방향을 평가한다.

관련성:

```text
0: 공유 판단 기준이 없음
1: 넓은 주제만 유사함
2: 일부 판단 기준을 공유함
3: 중요한 판단 기준을 공유함
4: 동일한 핵심 trade-off를 다룸
```

방향:

```text
SAME: 공유 판단축에서 같은 방향
OPPOSITE: 공유 판단축에서 반대 방향
MIXED: 여러 판단축이 섞여 있음
UNCLEAR: 관련성이 낮거나 판단 불가
```

Jev 요청 예시:

```json
{
  "model": "jev-1.13.0",
  "state": {
    "task": "두 선택지가 개인의 의사결정 경향을 해석할 때 어떤 관계인지 평가한다.",
    "rules": [
      "표현의 유사성이 아니라 선택에 필요한 판단 기준을 비교한다.",
      "질문에 없는 동기나 성격을 추정하지 않는다.",
      "서로 다른 상황이면 관련성을 낮춘다.",
      "여러 판단축이 섞였으면 MIXED를 선택한다."
    ],
    "left": {
      "node_id": "q001:o2",
      "question": "새로운 업무를 고른다면?",
      "all_options": ["익숙한 업무", "어려운 신규 프로젝트", "보상이 높은 업무", "부담이 적은 업무"],
      "focal_option": "어려운 신규 프로젝트"
    },
    "right": {
      "node_id": "q048:o3",
      "question": "이직할 회사를 선택할 때 무엇을 우선하나요?",
      "all_options": ["안정성", "높은 연봉", "성장 기회", "짧은 출퇴근"],
      "focal_option": "성장 기회"
    }
  },
  "questions": {
    "relevance": {
      "type": "score",
      "instructions": "두 선택이 동일하거나 비교 가능한 판단 기준을 공유하는 정도를 평가하라.",
      "criteria": [
        "공유 판단 기준이 없다.",
        "넓은 주제만 비슷하다.",
        "일부 판단 기준을 공유한다.",
        "중요한 판단 기준을 공유한다.",
        "동일한 핵심 trade-off를 다룬다."
      ]
    },
    "direction": {
      "type": "choice",
      "instructions": "공유 판단 기준에서 두 선택의 방향을 분류하라.",
      "criteria": {
        "SAME": "같은 방향의 선호를 나타낸다.",
        "OPPOSITE": "반대 방향의 선호를 나타낸다.",
        "MIXED": "일부는 같고 일부는 반대다.",
        "UNCLEAR": "관련성이 낮거나 판단할 수 없다."
      }
    }
  }
}
```

응답에서 다음 값을 저장한다.

```text
p_relevant = P(score=3) + P(score=4)

support_weight  = p_relevant × p_same
opposite_weight = p_relevant × p_opposite
mixed_weight    = p_relevant × p_mixed
```

Jev 모델은 `latest` 별칭 대신 명시적인 버전을 고정한다.

## 1.7 저장

Supabase 검색용 데이터:

```text
option_nodes
option_edges
```

R2 원본 산출물:

```text
preprocessing/
└── default-v1/
    ├── questionnaire.json
    ├── option-documents.jsonl
    ├── embeddings.parquet
    ├── jev-results.parquet
    └── manifest.json
```

## 1.8 실행 명령

```bash
uv run python -m preprocessing validate
uv run python -m preprocessing embed
uv run python -m preprocessing candidates --top-k 50
uv run python -m preprocessing evaluate-jev
uv run python -m preprocessing publish
```

---

# 영역 2. 사용자 질문 제공 및 답변 수집

## 2.1 목적

사용자가 300문항을 여러 번에 나누어 응답할 수 있도록 하고, 문항별로 안전하게 저장한다.

## 2.2 화면 흐름

```text
로그인
  → 질문지 설명
  → 문항 1개씩 표시
  → 문항별 자동 저장
  → 중단 가능
  → 마지막 위치부터 재개
  → 전체 답변 검토
  → 최종 제출
```

기본 UX:

- 한 화면에 질문 하나를 표시한다.
- 선택지는 큰 카드 또는 라디오 버튼으로 제공한다.
- 이유와 선택 변경 조건은 선택 입력으로 받는다.
- 상단에 현재 문항 번호와 진행률을 표시한다.
- `다음`을 누를 때 서버에 즉시 저장한다.
- 저장 실패 시 로컬에 임시 보관하고 재시도한다.
- 모바일 화면을 우선한다.
- 미응답 문항은 최종 제출 전에 보여준다.

## 2.3 답변 저장 API

```http
PUT /functions/v1/questionnaire/answers
```

```json
{
  "questionnaire_id": "default-v1",
  "question_id": "q034",
  "selected_option_id": "q034:o2",
  "reason": "새로운 경험을 하고 싶다.",
  "condition": "가족 시간이 크게 줄어들면 선택하지 않는다."
}
```

응답:

```json
{
  "answer_id": "answer_uuid",
  "revision": 2,
  "saved_at": "2026-09-29T15:03:00Z"
}
```

답변은 `(user_id, questionnaire_id, question_id)` 기준으로 upsert한다.

## 2.4 질문지 완료

```http
POST /functions/v1/questionnaire/complete
```

서버 검증:

- 300개 질문에 모두 답했는지 확인한다.
- 선택지가 해당 질문에 속하는지 확인한다.
- 현재 답변 revision을 고정한다.
- 동일 revision의 프로필 작업이 이미 있는지 확인한다.

응답:

```json
{
  "questionnaire_status": "COMPLETED",
  "profile_status": "QUEUED",
  "answer_revision": 300
}
```

## 2.5 데이터 모델

```text
questionnaire_sessions
- id
- user_id
- questionnaire_id
- status
- answered_count
- current_question_id
- answer_revision
- completed_at

answers
- id
- user_id
- questionnaire_id
- question_id
- selected_option_id
- reason
- condition
- revision
- updated_at
```

## 2.6 웹 경로

```text
/onboarding
/questionnaire/[questionnaireId]
/questionnaire/[questionnaireId]/review
/questionnaire/[questionnaireId]/complete
```

---

# 영역 3. 질문 결과 기반 전역 프로필 생성

## 3.1 목적

사용자의 300개 선택을 모든 추론 요청에 항상 포함할 수 있는 짧은 `PROFILE.md`로 변환한다. 프로필은 사용자에 대한 절대적 사실이 아니라 반복 선택에서 관찰된 판단 경향이다.

## 3.2 실행 흐름

```text
질문지 완료
  → profile_jobs 생성
  → Lambda 비동기 실행
  → 사용자 선택 300개 조회
  → 공통 그래프에 선택 투영
  → 군집·긴장·조건 추출
  → LLM profile.json 생성
  → Jev 주장 검증
  → PROFILE.md 렌더링
  → R2 저장
```

## 3.3 사용자 선택 그래프 투영

사용자가 고른 선택지 ID 집합을 `U`라고 한다.

```text
U = {q001:o2, q002:o4, ...}
```

공통 그래프에서 양쪽 노드가 모두 `U`에 포함된 edge를 가져온다.

```text
support edges:
support_weight가 높은 관계

tension edges:
opposite_weight가 높은 관계

mixed edges:
mixed_weight가 높은 관계
```

사용자별 `300C2` 관계를 다시 Jev로 평가하지 않는다.

## 3.4 군집과 대표 답변

support edge를 기준으로 선택 노드를 군집화한다. 임베딩은 그래프 연결이 약하거나 누락된 노드를 보충한다.

군집별 대표 답변:

- 그래프 중심성이 높은 답변 3개
- 임베딩 중심에 가까운 답변 2개
- 사용자가 이유를 작성한 답변
- 사용자가 조건을 작성한 답변
- tension 또는 mixed 관계가 있는 답변

군집 이름은 미리 정의하지 않고 LLM이 증거를 바탕으로 생성한다.

## 3.5 프로필 증거 패킷

```json
{
  "user_id": "user_uuid",
  "questionnaire_version": "default-v1",
  "answer_revision": 300,
  "answers": [
    {
      "question_id": "q012",
      "question": "새로운 업무를 고른다면?",
      "selected_option": "어려운 신규 프로젝트",
      "other_options": ["익숙한 업무", "보상이 높은 업무", "부담이 적은 업무"],
      "reason": "새로운 기술을 배우고 싶다.",
      "condition": "가족 돌봄 시간이 부족해지면 어렵다."
    }
  ],
  "communities": [
    {
      "id": "community_01",
      "member_question_ids": ["q012", "q048", "q091"],
      "representative_question_ids": ["q012", "q048"],
      "support_edge_count": 14,
      "mean_support_weight": 0.83
    }
  ],
  "tensions": [
    {
      "left_question_id": "q012",
      "right_question_id": "q077",
      "opposite_weight": 0.81
    }
  ]
}
```

## 3.6 프로필 생성 시스템 프롬프트

```text
당신은 사용자의 의사결정 프로필을 작성하는 분석기다.

목표:
사용자가 완료한 질문지와 사전 계산된 관계 증거를 바탕으로,
향후 새로운 선택 질문을 예측할 때 사용할 안정적인 전역 프로필을 만든다.

규칙:
1. 사용자가 실제로 선택하거나 직접 작성한 이유와 조건만 사실로 취급한다.
2. 선택하지 않은 선택지는 해당 비교에서 선택되지 않았다는 의미일 뿐,
   사용자가 항상 싫어한다는 뜻이 아니다.
3. 그래프 관계는 해석 보조 자료이며 확정 사실이 아니다.
4. 반복 근거가 있는 경향과 한 번만 나타난 선택을 구분한다.
5. 반대되는 답변을 억지로 하나의 성격으로 통합하지 않는다.
6. 상황이 다르면 모순이 아니라 context switch로 기록할 수 있다.
7. 성격 유형, 정신 상태, 건강 상태, 인구통계 정보를 추측하지 않는다.
8. 모든 주장에는 evidence_question_ids를 첨부한다.
9. 근거가 부족한 내용은 unknowns에 기록한다.
10. 직접 관련된 원본 문답이 이 프로필보다 우선한다.

출력:
제공된 JSON Schema를 정확히 따르는 JSON만 반환한다.
```

## 3.7 profile.json 구조

```json
{
  "summary": "string",
  "decision_principles": [
    {
      "id": "string",
      "title": "string",
      "observation": "string",
      "scope": ["string"],
      "strength": "strong | medium",
      "evidence_question_ids": ["string"],
      "counterevidence_question_ids": ["string"]
    }
  ],
  "priority_relations": [
    {
      "preferred": "string",
      "over": "string",
      "scope": "string",
      "strength": "strong | medium",
      "evidence_question_ids": ["string"]
    }
  ],
  "constraints": [
    {
      "condition": "string",
      "effect_on_choice": "string",
      "source": "user_stated | inferred_from_repeated_choices",
      "evidence_question_ids": ["string"]
    }
  ],
  "context_switches": [],
  "tensions": [],
  "unknowns": [],
  "generation_metadata": {
    "questionnaire_version": "string",
    "graph_version": "string",
    "embedding_model": "string",
    "jev_model": "string",
    "profile_model": "string"
  }
}
```

## 3.8 Jev 주장 검증

LLM이 만든 각 주장에 대해 다음을 평가한다.

```text
supported:
인용한 원본 답변이 주장을 충분히 뒷받침하는가?

overgeneralized:
제한된 선택을 사용자의 보편적인 성격으로 확대했는가?
```

초기 포함 기준 예시:

```text
P(supported) >= 0.75
P(overgeneralized) <= 0.25
```

실제 threshold는 사람이 라벨링한 검증 데이터로 조정한다.

## 3.9 PROFILE.md

검증된 `profile.json`을 코드로 Markdown으로 렌더링한다.

```md
---
profile_version: 1
questionnaire_version: default-v1
graph_version: graph-jev-1.13.0-v1
answer_revision: 300
---

# User Decision Profile

## 사용 규칙

- 직접 관련된 원본 답변이 이 프로필보다 우선한다.
- 사용자가 직접 작성한 조건은 일반 경향보다 우선한다.
- 선택하지 않은 선택지를 고정적인 비선호로 해석하지 않는다.
- 근거 없는 성격이나 동기를 추가하지 않는다.

## 전체 요약

성장과 새로운 경험을 선호하는 경향이 있지만 가족 돌봄 시간과
생활 안정성을 중요한 조건으로 둔다.

## 주요 판단 원칙

### 성장 가능성 대 익숙함

- 관찰: 업무와 학습에서는 성장 가능성을 우선하는 경향이 있다.
- 강도: 강함
- 근거: `q012`, `q048`, `q091`
- 반대 근거: `q077`

## 중요한 조건

- 가족 돌봄 시간이 크게 줄어들면 선택이 달라질 수 있다.
- 근거: `q012`

## 아직 알 수 없는 사항

- 성장 기회와 가족 시간이 직접 충돌할 때의 최종 우선순위
```

모든 요청에 포함하므로 2,000~3,000 토큰 이내로 제한한다.

## 3.10 저장

```text
users/
└── {user_id}/
    └── profiles/
        └── {profile_version}/
            ├── profile.json
            └── PROFILE.md
```

DB:

```text
profile_jobs
- id
- user_id
- answer_revision
- status
- attempts
- error
- created_at
- completed_at

user_profiles
- id
- user_id
- version
- answer_revision
- summary
- r2_json_key
- r2_markdown_key
- graph_version
- embedding_model
- jev_model
- profile_model
- created_at
```

`(user_id, answer_revision)`에 unique 제약을 두어 Lambda 중복 호출로 동일 프로필이 두 번 생성되지 않게 한다.

---

# 영역 4. 디지털 클론과 채팅

## 4.1 목적

사용자가 자유롭게 질문하면 일반적인 조언이 아니라 사용자의 과거 선택과 조건을 바탕으로 사용자가 할 법한 답변을 생성한다.

## 4.2 요청 처리 흐름

```text
사용자 메시지
  → 질문 정규화
  → 새 질문 임베딩
  ├─ 사용자 문답 top 20
  └─ 공통 선택지 top 10
  → 공통 그래프 한 단계 확장
  → 사용자가 실제 선택한 노드만 필터
  → 후보 통합·중복 제거
  → Jev 유용성 평가 및 역할 분류
  → 근거 8~12개 선정
  → PROFILE.md + 질문 + 원본 근거
  → 최종 LLM
```

## 4.3 자유 질문 정규화

예시 입력:

```text
나 이직하는 게 좋을까?
```

내부 검색 표현:

```json
{
  "decision_question": "현재 직장을 유지하는 것과 이직하는 것 중 사용자는 무엇을 선택할 가능성이 높은가?",
  "options": [
    { "id": "STAY", "text": "현재 직장을 유지한다." },
    { "id": "MOVE", "text": "새 직장으로 이직한다." }
  ],
  "known_facts": [],
  "unknown_facts": [
    "현재 직장의 만족도",
    "이직할 회사의 조건"
  ],
  "search_queries": [
    "직업 안정성과 성장 기회",
    "새로운 환경과 익숙한 환경",
    "연봉과 근무 조건"
  ]
}
```

## 4.4 후보 수집

```text
사용자 원본 문답 임베딩 검색: top 20
공통 선택지 검색: 선택지마다 top 10
전체 그래프 시작점: 3개
그래프 이웃: 시작점당 최대 5개
최종 후보: 중복 제거 후 약 30~50개
```

임베딩으로 직접 찾은 사용자 문답은 그래프 연결이 없어도 후보에서 유지한다.

## 4.5 Jev 재정렬

후보마다 다음을 평가한다.

```text
usefulness:
0: 관련 없음
1: 주제만 유사
2: 약한 배경 근거
3: 직접적인 선택 근거
4: 결론을 바꾸는 핵심 조건

role:
DIRECT_SUPPORT
COUNTEREVIDENCE
CONDITION
BACKGROUND
IRRELEVANT
```

최종 근거 구성:

```text
직접 근거       3~4개
조건            2~3개
반대 근거       1~2개
배경            1~2개
총합            8~12개
```

## 4.6 최종 LLM 시스템 프롬프트

```text
당신은 일반적인 조언자가 아니라 제공된 기록을 바탕으로
사용자가 할 법한 선택과 답변을 예측하는 디지털 클론이다.

근거 우선순위:
1. 현재 질문과 직접 관련된 원본 문답
2. 사용자가 직접 작성한 이유와 조건
3. 전역 사용자 프로필
4. 공통 선택지 그래프의 관계

규칙:
- 직접 근거가 프로필과 충돌하면 직접 근거를 우선한다.
- 사용자에게 없는 경험, 성격, 사실을 만들어내지 않는다.
- 반대 근거와 선택을 바꾸는 조건을 함께 검토한다.
- 정보가 부족하면 확실한 척하지 않는다.
- 사용자의 말투보다 판단 경향을 재현한다.
- 사용한 원본 근거는 evidence_id로 표시한다.
```

## 4.7 최종 요청 데이터

```json
{
  "profile": "...PROFILE.md...",
  "conversation": [
    {
      "role": "user",
      "content": "나 이직하는 게 좋을까?"
    }
  ],
  "normalized_decision": {
    "question": "현재 직장을 유지하는 것과 이직하는 것 중 무엇을 선택할 가능성이 높은가?",
    "options": ["현재 직장 유지", "이직"],
    "unknown_facts": ["이직할 회사의 조건"]
  },
  "evidence": [
    {
      "evidence_id": "q012",
      "role": "DIRECT_SUPPORT",
      "question": "새로운 업무를 고른다면?",
      "selected": "어려운 신규 프로젝트",
      "reason": "새로운 기술을 배우고 싶다.",
      "condition": "가족 돌봄 시간이 부족해지면 어렵다."
    }
  ]
}
```

출력:

```json
{
  "answer": "새 직장에서 성장 기회가 분명하다면 이직 쪽을 선택할 가능성이 높습니다.",
  "predicted_choice": "이직",
  "confidence": "medium",
  "supporting_evidence_ids": ["q012", "q048"],
  "counterevidence_ids": ["q077"],
  "decisive_conditions": [
    "가족 돌봄 시간이 유지되는지",
    "최소 수입 기준을 충족하는지"
  ],
  "follow_up_question": "이직할 회사의 근무 시간과 연봉은 현재 기준을 충족하나요?"
}
```

## 4.8 채팅 데이터

```text
chat_sessions
- id
- user_id
- profile_version
- created_at

chat_messages
- id
- session_id
- role
- content
- metadata
- created_at
```

채팅 기록은 자동으로 프로필에 반영하지 않는다. 사용자가 명시적으로 반영을 요청할 때만 새 답변 revision으로 저장한다.

## 4.9 웹 경로

```text
/clone
/clone/chat/[sessionId]
```

---

# 영역 5. 사용자와 디지털 클론의 유사도 평가

## 5.1 목적

새로운 질문에 대해 실제 사용자와 디지털 클론이 얼마나 비슷하게 답하는지 측정한다.

## 5.2 누출 방지 원칙

사용자 답변은 클론 예측이 완료되기 전까지 클론 요청에 포함하지 않는다.

```text
평가 질문 제공
  → 사용자가 답변
  → 사용자 답변 비공개 저장
  → 같은 질문을 디지털 클론에 요청
  → 클론 답변 저장
  → 두 답변 비교
```

평가 질문은 프로필 생성용 300문항과 분리한다.

```text
question_source = EVALUATION

평가 답변은 예측 완료 전 다음에서 제외:
- 전역 프로필
- 사용자 문답 벡터 검색
- 채팅 메모리
- 그래프의 사용자 선택 필터
```

## 5.3 사용자 답변

```json
{
  "evaluation_question_id": "eval_001",
  "selected_option_id": "B",
  "reason": "성장 가능성이 더 중요하다.",
  "condition": "연봉 차이가 더 커지면 A를 선택할 수 있다."
}
```

## 5.4 클론 답변

```json
{
  "evaluation_question_id": "eval_001",
  "selected_option_id": "B",
  "confidence": "medium",
  "reason": "과거 답변에서 성장 가능성을 반복적으로 우선했다.",
  "decisive_conditions": ["연봉 차이", "가족 돌봄 시간"],
  "evidence_ids": ["q012", "q048", "q077"]
}
```

## 5.5 평가 지표

### 선택 일치율

```text
choice_accuracy = 일치한 질문 수 / 전체 평가 질문 수
```

### Top-2 일치율

선택지가 많을 때 사용자의 선택이 클론의 상위 2개 예측 안에 들어간 비율이다.

### 이유 유사도

```text
0: 판단 기준이 다름
1: 일부 관련 있지만 핵심 이유가 다름
2: 핵심 판단 기준이 유사함
3: 이유와 우선순위가 거의 동일함
```

### 조건 발견률

사용자가 작성한 선택 변경 조건을 클론도 발견했는지 평가한다.

### 확신도별 정확도

```text
confidence=high인 답변 정확도
confidence=medium인 답변 정확도
confidence=low인 답변 정확도
```

### 영역별 정확도

```text
직업·성장
재정·안정
가족·시간
대인관계
위험 감수
생활 방식
학습
```

## 5.6 결과 표현

```text
현재 평가 질문 50개 중 38개에서 같은 선택을 예측했습니다.

선택 일치율        76%
이유 유사도        68%
조건 발견률        61%

직업·성장          88%
재정·안정          81%
가족·시간          74%
대인관계            52%
```

`사용자를 76% 이해했다`고 표현하지 않고 측정한 질문 범위를 함께 표시한다.

## 5.7 구성요소 기여도 평가

다음 네 설정을 같은 평가 질문으로 비교한다.

```text
A. 임베딩 RAG만
B. 임베딩 RAG + 그래프
C. 임베딩 RAG + 프로필
D. 임베딩 RAG + 그래프 + 프로필
```

그래프와 프로필은 실제 새 질문 정확도를 개선하는 경우에만 유지한다.

## 5.8 데이터 모델

```text
evaluation_sets
- id
- name
- questionnaire_version

evaluation_questions
- id
- evaluation_set_id
- question
- options
- category

evaluation_runs
- id
- user_id
- profile_version
- graph_version
- model
- status
- created_at

evaluation_answers
- id
- run_id
- question_id
- actor
- selected_option_id
- reason
- conditions
- confidence
- evidence_ids
- created_at
```

`actor` 값:

```text
USER
CLONE
```

## 5.9 웹 경로

```text
/evaluation
/evaluation/[runId]/question/[index]
/evaluation/[runId]/result
```

---

# 4. Lambda 및 API 구성

초기에는 세 개의 Lambda 영역으로 묶는다.

```text
questionnaire
- 답변 저장
- 진행 상태 조회
- 질문지 완료

profile
- 프로필 생성
- 프로필 상태 조회

clone
- 채팅
- 평가 실행
- 평가 결과 조회
```

상세 엔드포인트:

```text
PUT  /questionnaire/answers
GET  /questionnaire/status
POST /questionnaire/complete

POST /profiles/generate
GET  /profiles/status
GET  /profiles/current

POST /clone/chat
POST /evaluations
POST /evaluations/{runId}/answers
GET  /evaluations/{runId}/result
```

---

# 5. 데이터베이스 구성

```text
auth.users

questionnaires
questions
options

option_nodes
option_edges

questionnaire_sessions
answers

profile_jobs
user_profiles

chat_sessions
chat_messages

evaluation_sets
evaluation_questions
evaluation_runs
evaluation_answers
```

주요 버전 필드:

```text
questionnaire_version
embedding_model
graph_version
jev_model
relationship_rubric_version
profile_model
profile_prompt_version
answer_revision
```

모든 산출물은 어떤 질문지, 모델, 프롬프트로 만들어졌는지 추적할 수 있어야 한다.

---

# 6. 프로젝트 폴더 구조

```text
digital-clone/
├── apps/
│   └── web/
│       ├── app/
│       │   ├── onboarding/
│       │   ├── questionnaire/
│       │   │   └── [questionnaireId]/
│       │   ├── profile/
│       │   ├── clone/
│       │   │   └── chat/
│       │   └── evaluation/
│       │       └── [runId]/
│       │
│       ├── components/
│       │   ├── questionnaire/
│       │   │   ├── QuestionCard.tsx
│       │   │   ├── OptionCard.tsx
│       │   │   └── ProgressBar.tsx
│       │   ├── chat/
│       │   │   └── ChatPanel.tsx
│       │   └── evaluation/
│       │       └── ResultChart.tsx
│       │
│       └── lib/
│           ├── api.ts
│           ├── supabase.ts
│           └── types.ts
│
├── functions/
│   ├── questionnaire/
│   │   └── handler.py
│   ├── profile/
│   │   ├── handler.py
│   │   ├── evidence.py
│   │   ├── schema.py
│   │   └── render.py
│   └── clone/
│       ├── handler.py
│       ├── retrieval.py
│       ├── rerank.py
│       ├── chat.py
│       └── evaluation.py
│
├── preprocessing/
│   ├── __main__.py
│   ├── validate.py
│   ├── documents.py
│   ├── embeddings.py
│   ├── candidates.py
│   ├── jev_graph.py
│   └── publish.py
│
├── shared/
│   ├── db.py
│   ├── r2.py
│   ├── embeddings.py
│   ├── jev.py
│   ├── llm.py
│   ├── schemas.py
│   └── prompts/
│       ├── profile_system.md
│       ├── chat_system.md
│       └── evaluation_judge.md
│
├── supabase/
│   └── migrations/
│
├── data/
│   ├── questionnaires/
│   │   └── default-v1.json
│   └── evaluation/
│       └── evaluation-v1.json
│
├── tests/
│   ├── test_candidates.py
│   ├── test_profile_evidence.py
│   └── test_evaluation_leakage.py
│
├── infrastructure/
│   └── template.yaml
│
├── pyproject.toml
├── pnpm-workspace.yaml
└── README.md
```

초기에는 `repository`, `domain`, `adapter`, `usecase` 같은 추가 계층을 만들지 않는다. 실제 파일이 커질 때 필요한 경계만 분리한다.

---

# 7. 구현 순서

## 1단계: 공통 데이터

```text
1. 질문지 JSON 정의
2. 질문지 검증기
3. Supabase 스키마
4. 선택지 문서 생성
5. 임베딩 생성
6. top 50 후보 생성
7. Jev 그래프 생성
8. R2 및 Supabase 배포
```

## 2단계: 사용자 질문지

```text
9. Next.js 질문 화면
10. 문항별 자동 저장
11. 중단 및 재개
12. 전체 검토
13. 질문지 완료 처리
```

## 3단계: 프로필

```text
14. 사용자 선택 그래프 투영
15. 프로필 evidence packet 생성
16. profile.json 생성
17. Jev 주장 검증
18. PROFILE.md 렌더링
19. R2 저장 및 상태 표시
```

## 4단계: 채팅

```text
20. 자유 질문 정규화
21. 사용자 문답 벡터 검색
22. 공통 그래프 확장
23. Jev 후보 재정렬
24. 최종 LLM 응답
25. 채팅 화면
```

## 5단계: 평가

```text
26. 별도 평가 질문지
27. 사용자 답변 비공개 저장
28. 클론 블라인드 예측
29. 선택·이유·조건 비교
30. 영역별 결과 화면
31. RAG·그래프·프로필 ablation 비교
```

---

# 8. 최소 검증 항목

## 전처리

- 같은 질문의 선택지끼리는 top 50 후보에서 제외되는가?
- `(A, B)`와 `(B, A)`가 중복 저장되지 않는가?
- 동일 버전 재실행에서 이미 성공한 Jev 결과를 재호출하지 않는가?
- 모델과 rubric 버전이 기록되는가?

## 질문지

- 새로고침 후 마지막 저장 문항부터 이어지는가?
- 같은 문항을 수정해도 답변이 중복 생성되지 않는가?
- 300개 미완료 상태에서 완료 처리되지 않는가?

## 프로필

- 동일 answer revision에서 프로필이 중복 생성되지 않는가?
- 모든 프로필 주장에 원본 question ID가 있는가?
- 직접 작성한 조건이 일반 경향보다 우선하는가?
- Jev 검증을 통과하지 못한 주장이 Markdown에서 제외되는가?

## 채팅

- 임베딩 직접 검색 결과가 그래프 연결이 없어도 후보에 남는가?
- 반대 근거와 조건이 최종 프롬프트에 포함되는가?
- 프로필과 직접 근거가 충돌할 때 직접 근거를 우선하는가?

## 평가

- 사용자 평가 답변이 클론 예측 전에 검색되지 않는가?
- 같은 평가 질문에 같은 프로필과 그래프 버전을 사용하는가?
- 선택 정확도와 이유 유사도를 분리해 계산하는가?
- A/B/C/D 구성을 동일 질문으로 비교하는가?

---

# 9. 초기 버전에서 제외할 것

- Neo4j 같은 별도 그래프 DB
- Pinecone, Qdrant 같은 별도 벡터 DB
- Redis 및 Celery
- 상시 실행 FastAPI 서버
- 마이크로서비스 분리
- 사용자별 `300C2` Jev 전수 평가
- 채팅 내용을 자동으로 사용자 프로필에 학습
- 근거 없는 성격 유형 및 심리 진단

이 요소들은 실제 평가에서 현재 구성이 부족하다는 증거가 생겼을 때만 추가한다.

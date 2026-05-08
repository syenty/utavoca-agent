# CLAUDE.md — utavoca-agent

This workspace is the **content pipeline** for [utavoca](../utavoca/), a Japanese vocabulary learning PWA.  
Its sole job: turn raw lyrics files into validated vocabulary data and load it into the utavoca Supabase database.

---

## What This Project Does

```
[일본어 전용 가사]
lyrics/raw/{아티스트}_{곡명}.txt
        ↓ lyrics-converter   (한글 발음 + 번역 추가)
        ↓ lyrics-reviewer ×N (clean 될 때까지 반복)
        ↓
lyrics/{아티스트}_{곡명}.txt  ← [3행 포맷 확정]
        ↓ vocab-extractor + summary-writer  (병렬)
        ↓ vocab-validator ×N               (clean 될 때까지 반복)
        ↓ db-operator                      (INSERT)
        ↓ archivist                        (mv archive/)
```

---

## Lyrics File Format

File naming: `{アーティスト名}_{曲名}.txt`  
Example: `幾田りら_Voyage.txt`

Each lyric is stored in **3-line groups**:

```
日本語の歌詞          ← Line 1: Japanese lyric
로마자 발음            ← Line 2: Romanized pronunciation
한국어 번역            ← Line 3: Korean translation

(blank line between groups)
```

Example:
```
寝ぼけ眼を擦って
네보케마나코오 코슷테
졸린 눈을 비비며
```

---

## 사용 예시 — 프롬프트

### 전체 실행

| 상황 | 프롬프트 |
|------|---------|
| lyrics/ 전체 처리 | `lyrics/ 처리해줘` |
| 처리 후 결과 요약 포함 | `lyrics/ 전부 처리하고 결과 요약해줘` |

### 부분 실행

| 상황 | 프롬프트 |
|------|---------|
| 특정 파일 1개만 | `lyrics/幾田りら_Voyage.txt 처리해줘` |
| 어휘 추출·검증만 (DB 저장 안 함) | `lyrics/XXX.txt 어휘 추출하고 검증까지만 해줘. DB는 건드리지 마` |
| 요약만 생성 | `lyrics/XXX.txt 곡 요약만 써줘` |
| 어휘 미리 보기 (커밋 없이) | `lyrics/XXX.txt 어휘 뽑아서 보여줘만 줘. 저장은 하지 마` |

### 파이프라인 실행 전 셋업 — 가사 포맷 변환

일본어 전용 가사 파일은 먼저 `lyrics-converter`로 변환한 뒤 파이프라인을 실행합니다.

| 상황 | 프롬프트 |
|------|---------|
| 특정 파일 변환 | `lyrics/raw/Novelbright_イマナンドモ.txt 변환해줘` |
| raw/ 전체 변환 | `lyrics/raw/ 전부 변환해줘` |
| 변환 후 바로 파이프라인 실행 | `lyrics/raw/XXX.txt 변환하고 바로 처리해줘` |

변환 결과는 `lyrics/{파일명}`에 저장되고 원본 `lyrics/raw/{파일명}`은 유지됩니다.

### 파이프라인 실행 전 셋업 — 아티스트 관리

가사 파일을 처리하기 전에 해당 아티스트가 DB에 등록되어 있어야 합니다.

| 상황 | 프롬프트 |
|------|---------|
| 아티스트 등록 여부 확인 | `아티스트 [이름] DB에 있는지 확인해줘` |
| 아티스트 신규 등록 | 아래 형식으로 요청 |

**아티스트 등록 프롬프트 형식:**
```
아티스트 추가해줘:
- name: 星野源            ← 일본어 (필수, 파일명 매칭 기준)
- name_en: Hoshino Gen    ← 영문/로마자 (검색용)
- name_ko: 호시노 겐      ← 한국어 (검색용)
- image_url: null         ← 없으면 null
```

### 주의

- DB INSERT는 되돌릴 수 없습니다. "저장은 하지 마" / "DB는 건드리지 마" 를 명시하면 Phase 3을 건너뜁니다.
- 특정 파일 지정 시 파일명을 정확히 입력하세요 (`lyrics/` 경로 포함).
- 아티스트 `name`은 가사 파일명의 아티스트 부분과 정확히 일치해야 합니다 (`{アーティスト名}_{曲名}.txt`).

---

## Agent Team

This project uses seven specialized subagents. Claude acts as **Orchestrator** and dispatches them.

**셋업 에이전트** (파이프라인 실행 전)

| Agent | 역할 | 툴 | 모델 |
|-------|------|----|------|
| `lyrics-converter` | 일본어 전용 가사 → 3행 포맷 변환 | Read, Write | Sonnet |
| `lyrics-reviewer` | 3행 포맷 가사 품질 검수·교정 (반복 실행) | Read, Write | Sonnet |
| `db-operator` | 아티스트 UUID 조회 + 아티스트/song INSERT | Read, Bash | Haiku |

**파이프라인 에이전트** (lyrics/ 처리)

| Agent | 역할 | 툴 | 모델 |
|-------|------|----|------|
| `vocab-extractor` | 가사에서 어휘 후보 추출 | Read | Sonnet |
| `summary-writer` | 한국어 곡 요약 생성 | Read | Sonnet |
| `vocab-validator` | 어휘 품질 규칙 검증·교정 (반복 실행) | Read | Sonnet |
| `archivist` | 처리 완료 파일 archive/ 이동 | Bash | Haiku |

---

## Conversion Pipeline Routing Rules

`lyrics/raw/` 파일 변환 요청이 오면 아래 순서로 에이전트를 라우팅합니다.

### Step 1 — 변환 (lyrics-converter)

```
Agent(lyrics-converter) ← lyrics/raw/{파일명} 경로 전달
```

`lyrics-converter`는 일본어 전용 파일을 읽어 한글 발음 + 한국어 번역을 추가한 3행 포맷으로 변환한 뒤 `lyrics/{파일명}`에 저장합니다.

### Step 2 — 검수 루프 (lyrics-reviewer)

`lyrics-reviewer`를 **status가 `"clean"`이 될 때까지** 반복 실행합니다.

```
loop:
  result ← Agent(lyrics-reviewer, lyrics/{파일명})
  if result.status == "clean": break
  # lyrics-reviewer가 직접 파일을 수정하므로 다음 라운드도 같은 경로 전달
→ 3행 포맷 확정
```

`lyrics-reviewer`는 교정이 필요한 경우 파일을 직접 수정하고 `{"status": "corrected"}`를 반환합니다.  
`"clean"`이 반환되면 루프를 종료합니다.

### Step 3 — 이후 처리 (선택)

변환·검수 완료 후 바로 파이프라인을 실행하도록 요청받은 경우:
```
→ 메인 파이프라인(Orchestration Routing Rules)으로 진행
   대상 파일: lyrics/{파일명}
```
"변환만 해줘" 요청이면 Step 2에서 종료합니다.

### 타임라인 요약 (변환 단계)
```
t=0  lyrics/raw/{파일명} 변환 요청
t=1  lyrics-converter 실행 → lyrics/{파일명} 생성
t=2  lyrics-reviewer 1회차 → {"status": "corrected"|"clean"}
t=K  lyrics-reviewer status=="clean" → 루프 종료
t=K+1  [선택] 메인 파이프라인 시작
```

---

## Orchestration Routing Rules

가사 파일 처리 요청이 오면 아래 순서로 에이전트를 라우팅합니다.

### Phase 0 — 파일 큐 구성
`lyrics/` 디렉토리를 스캔해 처리할 파일 목록을 만들고, **파일 하나씩 순차 처리**합니다.
파일이 여러 개여도 동시에 처리하지 않습니다 — 한 파일의 전체 파이프라인이 완료(아카이브)된 후 다음 파일로 넘어갑니다.

```
queue ← ls lyrics/*.txt          # 처리 대상 전체 목록 확정
total ← len(queue)
for i, file in enumerate(queue):
  print(f"[{i+1}/{total}] {file} 처리 시작")
  → Phase 1–4 실행
  print(f"[{i+1}/{total}] {file} 완료")
```

파일이 0개면 "처리할 파일이 없습니다"를 출력하고 종료합니다.

### Phase 1 — 병렬 처리 (파일당, 동시 실행)
`vocab-extractor`와 `summary-writer`는 같은 파일을 독립적으로 읽으므로 **동시에 spawn**합니다.

```
Agent(vocab-extractor, run_in_background=true)  ←┐ 동시
Agent(summary-writer,  run_in_background=true)  ←┘ 실행
```

### Phase 2 — 반복 검증 루프 (vocab-extractor 완료 후)
`vocab-validator`를 **status가 `"clean"`이 될 때까지** 반복 실행합니다.

```
vocabs ← vocab-extractor 결과
loop:
  result ← Agent(vocab-validator, vocabs)
  if result.status == "clean": break
  vocabs ← result.vocabs   # 교정된 배열로 다음 라운드
→ vocabs 확정
```

`vocab-validator`는 매 실행마다 `{"status": "clean"|"corrected", "vocabs": [...]}` 형식으로 반환합니다.
`"corrected"`가 반환되면 교정된 배열을 다음 입력으로 재사용합니다.
`"clean"`이 반환되면 루프를 종료하고 즉시 Phase 3으로 진행합니다.

### Phase 3 — DB 쓰기 (검증 완료 후 자동 진행)
검증 루프가 `"clean"`으로 종료되면 승인 대기 없이 즉시 실행합니다.

```
Agent(db-operator)  ← 아티스트명, 곡명, summary, vocabs 전달
```
`db-operator`는 먼저 아티스트 UUID를 조회한 뒤 song INSERT를 실행합니다.
INSERT 실패 시 해당 파일을 건너뛰고 다음 파일로 진행합니다 — 실패 내역은 최종 결과에 기록합니다.

### Phase 4 — 아카이브 (INSERT 성공 확인 후)
`db-operator`가 song UUID를 반환한 경우에만 실행합니다.
```
Agent(archivist)  ← 파일명 + song UUID(INSERT 성공 증거) 전달
```

### 타임라인 요약 (파일 N개)
```
t=0    lyrics/ 스캔 → queue = [file1, file2, ..., fileN]

── file1 처리 ──────────────────────────────────────────
t=1    [병렬] vocab-extractor + summary-writer 시작
t=2    vocab-extractor 완료 → vocab-validator 1회차
t=3    summary-writer 완료 (대기)
t=K    vocab-validator status=="clean" → 루프 종료
t=K+1  db-operator: UUID 조회 → INSERT
t=K+2  INSERT 성공 → archivist: mv archive/

── file2 처리 (file1 완료 후 시작) ─────────────────────
t=K+3  [병렬] vocab-extractor + summary-writer 시작
...    (반복)

── 전체 완료 ────────────────────────────────────────────
t=Z    성공 M개 / 실패 L개 요약 출력
```

---

## Processing a Lyrics File

When asked to process a lyrics file, follow these steps in order.

### Step 1 — Extract Vocabulary

Read the lyrics file and extract vocabulary words suitable for Japanese learners.

**Selection criteria:**
- Content words only: nouns, verbs (dictionary form), adjectives, adverbs
- Skip particles (は, が, を, に…), conjunctions, fillers
- Skip hiragana-only words under 2 mora (too simple)
- Prefer words that appear in the lyrics in kanji or katakana form
- Target 30–60 words per song

**Output format for each word:**
```
name: {Japanese word — kanji/katakana preferred}
meaning: {Korean meaning}
pronunciation: {romaji — Hepburn system, lowercase}
```

Example:
```
name: 夢
meaning: 꿈
pronunciation: yume
```

### Step 2 — Generate Song Summary

Write a short summary of the song in Korean, following the utavoca format exactly:

```
{3–4 sentence description of the song's theme and mood in Korean}

🎭 분위기: {mood1}, {mood2}, {mood3}

🔑 키워드: {word1} ({Korean meaning}), {word2} ({Korean meaning}), ...

📖 테마: {theme1}, {theme2}, {theme3}
```

Example:
```
아직 어중간한 꿈을 품고 거친 파도를 향해 나아가는 여정을 노래한 곡입니다. 두려움과 불안을 껴안으면서도 멈추지 않겠다는 강한 의지를 담았습니다. 곁에서 함께해 주는 사람의 존재가 앞으로 나아갈 힘이 된다는 메시지를 전합니다. 한 번뿐인 인생, 후회 없는 항해를 하고 싶다는 마음이 곡 전반에 흐릅니다.

🎭 분위기: 도전적, 감동적, 희망적

🔑 키워드: 夢 (꿈), 荒波 (거친 파도), 航海 (항해)

📖 테마: 성장, 우정, 도전
```

### Step 3 — Build the Supabase INSERT

Look up the artist UUID first, then build the SQL:

```sql
-- 1. Find artist UUID
SELECT id FROM artists WHERE name = 'アーティスト名';

-- 2. Insert song
INSERT INTO songs (artist_id, title, summary, vocabs)
VALUES (
  '{artist-uuid}',
  '曲タイトル',
  '{summary from Step 2}',
  '[
    {"name":"夢","meaning":"꿈","pronunciation":"yume"},
    {"name":"荒波","meaning":"거친 파도","pronunciation":"aranami"}
  ]'
);
```

**Supabase connection:** `.env` 파일(프로젝트 루트)에 `SUPABASE_URL`과 `SUPABASE_SERVICE_ROLE_KEY`가 있습니다.  
INSERT 작업은 반드시 service role key를 사용해야 합니다. `db-operator` 에이전트가 이를 처리합니다.

### Step 4 — Archive the Lyrics File

After successful INSERT, move the file to `archive/`:

```bash
mv lyrics/{アーティスト名}_{曲名}.txt archive/
```

---

## Vocabulary Quality Rules

These rules apply during both extraction and review:

| Rule | Detail |
|------|--------|
| Pronunciation format | Hepburn romaji, lowercase, no macrons (use `aa`/`oo`/`uu` for long vowels) |
| Meaning format | Natural Korean, not dictionary-literal |
| Kanji preferred | Use kanji form when it appears in the lyrics (not hiragana reading) |
| No duplicates | If a word appears multiple times, include it once |
| Verb form | Dictionary form (辞書形): 走る not 走って |
| Adjective form | Plain form: 無謀な not 無謀に (when both appear, use the more common form) |

---

## File Layout

```
utavoca-agent/
├── CLAUDE.md                        # This file
├── .claude/
│   └── agents/                      # 서브에이전트 정의 (7개)
├── lyrics/
│   ├── raw/                         # 일본어 전용 원문 (변환 전)
│   │   └── {アーティスト名}_{曲名}.txt
│   └── {アーティスト名}_{曲名}.txt   # 3행 포맷 (파이프라인 대기)
└── archive/                         # 처리 완료 파일
    └── {アーティスト名}_{曲名}.txt
```

---

## Relationship to utavoca

The utavoca app lives at `../utavoca/`. Refer to `../utavoca/CLAUDE.md` for:
- Full database schema details
- Supabase client patterns
- The `Vocab` JSONB interface definition
- RLS and security notes

Never modify the utavoca source code from this workspace — this project only produces SQL to run against the database.

---

## Quick Reference — utavoca DB Schema

```sql
-- songs table (relevant columns)
songs (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  artist_id   UUID REFERENCES artists(id),
  title       TEXT NOT NULL,
  summary     TEXT,
  vocabs      JSONB  -- [{name, meaning, pronunciation}]
)

-- vocab JSONB shape
{
  "name": "愛",           -- Japanese (kanji/katakana)
  "meaning": "사랑",      -- Korean
  "pronunciation": "ai"   -- Hepburn romaji
}
```

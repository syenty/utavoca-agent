# CLAUDE.md — utavoca-agent

This workspace is the **content pipeline** for [utavoca](../utavoca/), a Japanese vocabulary learning PWA.  
Its sole job: turn raw lyrics files into validated vocabulary data and load it into the utavoca Supabase database.

---

## What This Project Does

```
[일본어 전용 가사 — 기본 입력 경로]
lyrics/raw/{아티스트} - {곡명}.txt
        ↓ lyrics-processor   (초벌 변환 → 자체 검토 → 최종 저장)
        ↓
lyrics/{아티스트} - {곡명}.txt  ← [3행 포맷 확정]
        ↓ vocab-extractor + summary-writer  (병렬)
        ↓ vocab-validator ×N               (clean 될 때까지 반복)
        ↓ db-operator                      (INSERT)
        ↓ archivist                        (mv archive/)
```

---

## Lyrics File Format

File naming: `{アーティスト名} - {曲名}.txt`  
Example: `幾田りら - Voyage.txt`

Each lyric is stored in **3-line groups**:

```
日本語の歌詞          ← Line 1: Japanese lyric
한글 발음              ← Line 2: Korean phonetic transcription
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

가사 파일은 `lyrics/raw/`에 두는 것이 기본입니다. `lyrics/ 처리해줘` 한 마디면 raw 변환부터 DB 저장까지 자동으로 진행됩니다.

| 상황 | 프롬프트 |
|------|---------|
| 전체 처리 (raw 변환 포함) | `lyrics/ 처리해줘` |
| 처리 후 결과 요약 포함 | `lyrics/ 전부 처리하고 결과 요약해줘` |

### 부분 실행

| 상황 | 프롬프트 |
|------|---------|
| raw 파일 1개만 | `lyrics/raw/幾田りら - Voyage.txt 처리해줘` |
| 어휘 추출·검증만 (DB 저장 안 함) | `lyrics/raw/XXX.txt 어휘 추출하고 검증까지만 해줘. DB는 건드리지 마` |
| 요약만 생성 | `lyrics/raw/XXX.txt 곡 요약만 써줘` |
| 어휘 미리 보기 (커밋 없이) | `lyrics/raw/XXX.txt 어휘 뽑아서 보여줘만 줘. 저장은 하지 마` |
| 변환만 (파이프라인 없이) | `lyrics/raw/XXX.txt 변환만 해줘` |

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
- 특정 파일 지정 시 파일명을 정확히 입력하세요 (`lyrics/raw/` 경로 포함).
- 아티스트 조회는 퍼지 검색(`search_artists_fuzzy`)을 사용합니다. 파일명의 아티스트 부분과 DB `name`이 완전히 같지 않아도 검색되지만, 유사도가 낮으면 오케스트레이터에 후보 목록이 보고되고 선택 대기가 발생합니다.

---

## Agent Team

This project uses six specialized subagents. Claude acts as **Orchestrator** and dispatches them.

**셋업 에이전트** (파이프라인 실행 전)

| Agent | 역할 | 툴 | 모델 |
|-------|------|----|------|
| `lyrics-processor` | 일본어 전용 가사 → 초벌 변환 → 자체 검토 → 최종 3행 포맷 저장 | Read, Write | Sonnet |
| `db-operator` | 아티스트 UUID 조회 + 아티스트/song INSERT | Read, Bash | Haiku |

**파이프라인 에이전트** (lyrics/ 처리)

| Agent | 역할 | 툴 | 모델 |
|-------|------|----|------|
| `vocab-extractor` | 가사에서 어휘 후보 추출 | Read | Sonnet |
| `summary-writer` | 한국어 곡 요약 생성 | Read | Sonnet |
| `vocab-validator` | 어휘 품질 규칙 검증·교정 (반복 실행) | Read | Sonnet |
| `archivist` | 처리 완료 파일 archive/ 이동 | Bash | Haiku |

---

## Orchestration Routing Rules

가사 파일 처리 요청이 오면 아래 순서로 에이전트를 라우팅합니다.

### Phase 0 — 파일 큐 구성
`lyrics/raw/`가 **기본 입력 경로**입니다. 두 디렉토리를 순서대로 스캔해 처리 큐를 구성합니다.

```
raw_files  ← ls lyrics/raw/*.txt   # 미변환 파일
ready_files ← ls lyrics/*.txt      # 이미 변환된 파일

queue ← raw_files + ready_files    # raw 먼저, 이미 변환된 것은 뒤에 붙임
```

파일이 0개면 "처리할 파일이 없습니다"를 출력하고 종료합니다.

큐를 확정한 뒤 **파일 하나씩 순차 처리**합니다 — 한 파일의 전체 파이프라인이 완료(아카이브)된 후 다음 파일로 넘어갑니다.

```
total ← len(queue)
for i, file in enumerate(queue):
  print(f"[{i+1}/{total}] {file} 처리 시작")
  if file in raw_files:
    → Phase 0.5 실행 후 Phase 1–4
  else:
    → Phase 1–4 실행
  print(f"[{i+1}/{total}] {file} 완료")
```

### Phase 0.5 — 변환 (raw 파일인 경우)
`lyrics/raw/`에서 온 파일은 파이프라인 진입 전에 변환합니다.

```
Agent(lyrics-processor) ← lyrics/raw/{파일명} 경로 전달
→ lyrics/{파일명} 생성 확인 후 Phase 1로 진행
```

변환 실패 시 해당 파일을 건너뛰고 다음 파일로 진행합니다.

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
`db-operator`는 먼저 퍼지 검색으로 아티스트 UUID를 조회한 뒤 song INSERT를 실행합니다.
유사도 1위 score < 0.8이면 상위 후보 3개를 보고하고 아티스트 선택을 대기합니다 — 선택 후 INSERT를 재개합니다.
INSERT 실패 시 해당 파일을 건너뛰고 다음 파일로 진행합니다 — 실패 내역은 최종 결과에 기록합니다.

### Phase 4 — 아카이브 (INSERT 성공 확인 후)
`db-operator`가 song UUID를 반환한 경우에만 실행합니다.
```
Agent(archivist)  ← 파일명 + song UUID(INSERT 성공 증거) 전달
```

### 타임라인 요약 (파일 N개)
```
t=0    lyrics/raw/ + lyrics/ 스캔 → queue = [raw1, raw2, ..., ready1, ...]

── raw1 처리 (미변환 파일) ──────────────────────────────
t=1    lyrics-processor 실행 → lyrics/{raw1} 생성
t=2    [병렬] vocab-extractor + summary-writer 시작
t=3    vocab-extractor 완료 → vocab-validator 1회차
t=4    summary-writer 완료 (대기)
t=K    vocab-validator status=="clean" → 루프 종료
t=K+1  db-operator: UUID 조회 → INSERT
t=K+2  INSERT 성공 → archivist: mv archive/

── raw2 처리 (raw1 완료 후 시작) ────────────────────────
t=K+3  lyrics-processor 실행 → lyrics/{raw2} 생성
...    (반복)

── 전체 완료 ────────────────────────────────────────────
t=Z    성공 M개 / 실패 L개 요약 출력
```

---

## Vocabulary Quality Rules

These rules apply during both extraction and review:

| Rule | Detail |
|------|--------|
| Pronunciation format | Hepburn romaji, lowercase, no macrons — おう → `ou`, おお → `oo`, うう → `uu`, ああ → `aa` |
| Meaning format | Natural Korean, not dictionary-literal |
| Kanji preferred | Use kanji form when it appears in the lyrics (not hiragana reading) |
| No duplicates | If a word appears multiple times, include it once |
| Verb form | Dictionary form (辞書形): 走る not 走って |
| Adjective form | Plain form: 無謀な not 無謀に (when both appear, use the more common form) |
| Count | 30–60 entries per song; trim lowest-value words if over 60 |

---

## File Layout

```
utavoca-agent/
├── CLAUDE.md                        # This file
├── .claude/
│   └── agents/                      # 서브에이전트 정의 (6개)
├── lyrics/
│   ├── raw/                         # 일본어 전용 원문 (변환 전)
│   │   └── {アーティスト名} - {曲名}.txt
│   └── {アーティスト名} - {曲名}.txt   # 3행 포맷 (파이프라인 대기)
└── archive/                         # 처리 완료 파일
    └── {アーティスト名} - {曲名}.txt
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

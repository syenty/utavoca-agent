# utavoca-agent

[utavoca](https://github.com/syenty/utavoca)의 콘텐츠 파이프라인 워크스페이스입니다.  
일본어 가사 파일을 받아 어휘를 추출·검증하고 Supabase 데이터베이스에 적재합니다.

## 파이프라인 흐름

```
lyrics/{아티스트}_{곡명}.txt
        ↓  어휘 추출 + 곡 요약 생성 (병렬)
        ↓  어휘 품질 검증 (clean 될 때까지 반복)
        ↓  Supabase INSERT
        ↓
   archive/{아티스트}_{곡명}.txt
```

## 에이전트 팀

Claude가 Orchestrator 역할을 맡아 5개의 전문 서브에이전트를 순서에 맞게 dispatch합니다.

| 에이전트 | 역할 |
|---------|------|
| `vocab-extractor` | 가사에서 학습용 어휘 추출 |
| `summary-writer` | 한국어 곡 요약 생성 |
| `vocab-validator` | 어휘 품질 검증 및 교정 (반복 실행) |
| `db-operator` | 아티스트 UUID 조회 + song INSERT |
| `archivist` | 처리 완료 파일 archive/ 이동 |

## 시작하기

### 1. 환경 변수 설정

```bash
cp .env.example .env
```

`.env`에 Supabase 접속 정보를 입력합니다:

```env
SUPABASE_URL=https://your-project-id.supabase.co
SUPABASE_KEY=your-anon-key
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
```

### 2. 가사 파일 준비

`lyrics/` 디렉토리에 파일을 추가합니다.

**파일명 형식:** `{アーティスト名}_{曲名}.txt`

**파일 포맷** — 3행 그룹:
```
日本語の歌詞
로마자 발음
한국어 번역

(빈 줄)
```

예시:
```
夢の欠片は眠ったまま
유메노 카케라와 네뭇타마마
꿈의 조각은 잠든 채 그대로
```

### 3. Claude Code에서 실행

```
lyrics/ 처리해줘
```

## 사용 예시

| 목적 | 프롬프트 |
|------|---------|
| 전체 처리 | `lyrics/ 처리해줘` |
| 특정 파일만 | `lyrics/幾田りら_Voyage.txt 처리해줘` |
| DB 저장 없이 어휘만 확인 | `lyrics/XXX.txt 어휘 뽑아서 보여줘만 줘. 저장은 하지 마` |
| 곡 요약만 | `lyrics/XXX.txt 곡 요약만 써줘` |
| 아티스트 등록 여부 확인 | `아티스트 [이름] DB에 있는지 확인해줘` |

## 디렉토리 구조

```
utavoca-agent/
├── .claude/
│   └── agents/          # 서브에이전트 정의 (5개)
├── lyrics/              # 처리 대기 가사 파일
├── archive/             # 처리 완료 가사 파일
├── .env                 # Supabase 접속 정보 (커밋 안 함)
└── CLAUDE.md            # 에이전트 운영 지침
```

## 관련 프로젝트

- **[utavoca](https://github.com/syenty/utavoca)** — 이 파이프라인이 데이터를 공급하는 일본어 학습 PWA

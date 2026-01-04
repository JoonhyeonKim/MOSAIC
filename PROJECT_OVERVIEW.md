# MOSAIC 프로젝트 문서 (As-Is / To-Be)

## 1) 프로젝트 개요
MOSAIC는 멀티 플랫폼 OSINT 데이터를 수집해 행동/사회적 신호를 구조화하고, 필요 시 로컬 LLM(Ollama)을 통해 분석 리포트를 생성하는 프레임워크다. 핵심 목적은 단순 데이터 집계가 아니라, 플랫폼 간 행동 신호를 비교/해석 가능한 형태로 제공하는 것이다.

## 2) 프로젝트 구조
```
MOSAIC/
├─ MOSAIC.py                 # 오케스트레이터(수집 → 선택적 분석)
├─ README.md                 # 사용법/개념 설명
├─ requirements.txt          # 수집 단계 의존성
├─ requirements_llm.txt      # LLM 분석 단계 의존성
├─ results/                  # 수집/분석 결과 저장 (git ignore 가정)
└─ modules/
   ├─ Collecte.py            # 수집 엔진 + UI + 플랫폼별 Extractor
   ├─ Analysis.py            # LLM 분석 인터랙티브 UI
   ├─ llm_backend.py         # LLM 백엔드(현재 Ollama 로컬)
   ├─ config.yaml.example    # API 키/설정 템플릿
   └─ prompts/               # 분석 프롬프트 템플릿(마크다운)
      ├─ PoC.md
      ├─ Threat_Intelligence.md
      ├─ Recruitment.md
      └─ OPSEC_audit.md
```

## 3) 작동 방식 (현재 As-Is)
### 3.1 실행 진입점
- `MOSAIC.py`가 전체 파이프라인 오케스트레이션을 수행.
- 실행 시 `Collecte.py`를 호출해 수집 단계 수행.
- 수집 종료 후, 사용자의 선택에 따라 `Analysis.py`로 로컬 LLM 분석 진행.

### 3.2 수집(Extraction) 단계
- `modules/Collecte.py` 내부에 UI 시스템과 플랫폼별 Extractor 클래스가 통합되어 있음.
- 지원 플랫폼(총 8개): GitHub, StackOverflow, YouTube, Bluesky, Mastodon, Reddit, Medium, Telegram.
- CLI 모드와 인터랙티브 모드를 모두 지원.
  - CLI: `--pattern`(사용자명/패턴)과 `--platforms`(숫자/이름/ALL) 지정
  - 인터랙티브: 메뉴에서 플랫폼 선택 → 사용자명 입력
- 결과는 `results/`에 JSON으로 저장.

### 3.3 분석(LLM) 단계
- `modules/Analysis.py`에서 프롬프트와 결과 JSON을 선택 → LLM 호출.
- LLM 백엔드: `modules/llm_backend.py`
  - 현재는 Ollama 로컬 서버만 지원 (`http://localhost:11434`)
  - Cloud 백엔드는 미구현
- 분석 결과는 `results/analysis_*.txt`로 저장.

### 3.4 설정/보안
- `modules/config.yaml.example`에 API 키/플랫폼 설정 템플릿 제공.
- 실제 키는 `modules/config.yaml`로 복사 후 사용.
- 수집 결과 및 분석 결과는 로컬 저장(외부 전송 없음).

## 4) As-Is 요약
### 강점
- 멀티 플랫폼 통합 수집과 일관된 JSON 출력.
- 수집-분석 파이프라인이 단순하고 명확함.
- 로컬 LLM 분석 옵션으로 프라이버시 우선 설계.

### 한계/리스크
- `config.yaml`의 일부 설정(레이트 리밋, 출력 설정 등)이 실제 로직에서 아직 사용되지 않음.
- Cloud LLM 백엔드 미구현.
- `README.md`에는 `mosaic.py`로 표기되어 있으나 실제 파일명은 `MOSAIC.py`로 대소문자 불일치.
- UI, 추출기, 오케스트레이션 로직이 `Collecte.py`에 혼재되어 구조가 비대함.
- 결과 스키마 표준화/버전 관리가 없음.

## 5) To-Be 제안
### 5.1 구조 개선
- `Collecte.py`를 다음으로 분리:
  - `ui/` (UI 컴포넌트)
  - `extractors/` (플랫폼별 수집기)
  - `orchestrator.py` (오케스트레이션)
- `config.yaml`의 설정을 실제 로직에 반영(레이트 리밋, 출력 경로, 제한 수 등).

### 5.2 실행/사용성 개선
- `README.md`의 실행 명령과 실제 파일명 정합성 맞추기.
- 결과 JSON 스키마 정의 및 버전 메타데이터 추가.
- 수집 실패/부분 성공 시 재시도 옵션 제공.

### 5.3 분석 기능 확장
- Cloud LLM 백엔드 구현(예: OpenAI/Anthropic) 및 키 관리 지원.
- 프롬프트 템플릿에 메타데이터(목적, 입력 기대치, 출력 형식) 정의.
- 분석 결과를 Markdown으로도 저장 옵션 제공.

### 5.4 테스트/품질
- 최소한의 통합 테스트(플랫폼별 API 호출 mocking)와 스키마 검증 추가.
- `results/` 샘플 출력 기반의 회귀 테스트 추가.

## 6) 한눈에 보는 파이프라인
```
사용자 실행
  └─ MOSAIC.py
      ├─ Collecte.py (수집: GitHub/StackOverflow/YouTube/...)
      │   └─ results/*.json 저장
      └─ Analysis.py (선택)
          ├─ prompts/*.md 선택
          └─ Ollama 호출 → results/analysis_*.txt 저장
```

# Portfolio Vault

인프라/DevOps 경력 기술 기록. Obsidian vault, GitHub 공개 저장소.
글쓰기 규칙은 전역 CLAUDE.md의 "Wiki / 포트폴리오 문서 작성"을 따름.

## 폴더 구조

- `IBM KTS/` — 고객사별 프로젝트 기록
- `제주항공/` — 시간순 번호(`NN-제목.md`) 프로젝트 기록
- `TelePIX/` — 도메인별 폴더(`01. infra` ~ `05. monitoring`)
  - `00. projects/`는 링크만 모으는 목차. 실제 문서는 도메인 폴더에 둠
- 각 폴더의 `00-index.md` — 문서 목록 표 (MOC)
- `서류/` — 이력서·경력기술서 등 개인 서류. gitignore 대상, 공개 금지

## 문서 추가·수정 시

- 문서 추가·삭제·이름 변경 시 해당 폴더 `00-index.md` 표와 `README.md` 폴더 구성의 건수를 함께 갱신
- 번호 체계 폴더(`제주항공/`, `IBM KTS/`)는 마지막 번호 다음으로 추가
- frontmatter `updated`는 수정한 날짜로 갱신
- git commit 하지 않음 — Obsidian 자동 백업이 "vault backup" 커밋으로 처리

## 지원 서류(`서류/`)로 옮길 때

- 수치는 반올림 (138대 → 약 140대), 한 불릿에 숫자 1~2개
- "하드닝" 대신 "OS 보안취약점 조치"
- 날짜·버전·등급 수 같은 고유값은 그대로

## 검수 에이전트 (`.claude/agents/`)

모두 읽기 전용, 지적 목록만 반환. 수정 반영은 메인 세션에서 함.

- `style-reviewer` — 문체·구조, AI 글쓰기 패턴
- `consistency-checker` — 문서 간 수치·기간 충돌, 깨진 링크, index/README 건수
- `security-reviewer` — 공개 전 민감 정보

수정 후 세 에이전트에 같은 대상(파일 또는 폴더)을 넘겨 병렬 검수.

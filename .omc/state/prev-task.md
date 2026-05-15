# pwno-mcp fork — Previous Task State

> 완료된 작업 아카이브. 최신 작업은 `current-task.md`.

---

## Fork bootstrap (2026-05-15) — 완료

**동기:** OmP의 challenge1 실측 후 사용자가 kernel CTF 로드맵 + pwno-mcp customization 결정. 단순 vanilla 추가 fix가 아니라 architecture 수준 변경 예정이라 별개 fork 채택.

**작업:**
- `cd ~/Tools && gh repo fork pwno-io/pwno-mcp --clone` → `youner119/pwno-mcp` 생성 + `~/Tools/pwno-mcp` clone
- upstream remote 자동 추가 (`git remote -v` 로 확인 가능)
- 로컬 `main` 브랜치는 `origin/main` 추적, `upstream/main`과 일치
- 디렉토리 패턴은 `~/Tools/binary_ninja_mcp` (기존 fork pattern)와 동일

**진단 / 발견:**
- upstream의 `feat/support-parallel-debug` 브랜치는 **이미 main에 머지됨** (PR #17). main에는 추가로:
  - `c678df1 fix(router): avoid event-loop blocking`
  - `2fb9e45 feat!(mcp): require explicit session_id`
  - `77cfbdb refactor: fastmcp-v3` (PR #18)
  - `6585ead refactor: move gdb/retdec backends to tools package`
- 즉 OmP가 본 Round 2 timeout이 vanilla main의 architecture 이슈일 가능성보다 **stale local docker image 가능성이 더 높음**. 검증 보류 (current-task.md의 pending decision #1).

**파일 추가:**
- `CLAUDE.md` (이 fork 컨텍스트)
- `.omc/state/current-task.md`, `.omc/state/prev-task.md`
- `.omc/decisions.md`
- `.omc/specs/` 디렉토리 (스펙 작성 예정 위치)

**OmP 측 변경:**
- `/mnt/D/Hack/oh-my-pwn/CLAUDE.md` 에 "pwno-mcp custom fork" 섹션 추가. binary_ninja_mcp 패턴과 평행.

**검증:** fork commits 0 (vanilla main 그대로). 사용자가 OmP에서 vanilla `ghcr.io/pwno-io/pwno-mcp:latest`를 그대로 쓰는 한 OmP 동작에 영향 0.

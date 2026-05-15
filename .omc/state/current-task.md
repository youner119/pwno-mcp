# pwno-mcp fork — Current Task State

> 이 fork는 OmP가 소비. 변경의 큰 그림은 `CLAUDE.md` 참조.
> Spec 1차 완료 → sub-task sequential 진행 단계.
> 완료된 sub-task는 `prev-task.md`로 이동.

**Last updated:** 2026-05-15 (post deep-interview, ambiguity 17%)
**Branch (현재):** `main` (vanilla 추적, 우리 commits 0)
**Branch (작업 예정):** `feat/per-docker-refactor` (sub-task 1에서 생성)

## Phase / 컨텍스트

**Active workstream:** **C1 Host pwno-mcp + Docker-as-tool refactor** (분기 A — pwno-mcp 호스트 Python process + 각 debug session = 별개 worker docker container).

**Spec:** `.omc/specs/deep-interview-c1-host-mcp-docker-tool.md` (deep-interview 결과 — 7 rounds, ambiguity 17%, threshold 20% 도달).

**Ship gate (commit 단위):** Minimal demo
- Phase 1a: 일반 MCP client (Claude Code/Codex 등) → 호스트 pwno-mcp 연결 + tool 목록 + read-only tool 1개 호출
- Phase 1b: `feat/per-docker-refactor` 브랜치에서 worker container spawn → 1 fixture ELF로 gdb-mi 1회 응답 → cleanup

## Resolved decisions (deep-interview 종결)

| # | Decision | 근거 (라운드) |
|---|----------|---------------|
| R0-R0 | 분기 A 채택 (호스트 pwno-mcp + worker container들), 분기 B 폐기 | R0 follow-up: "로컬 mcp가 docker들 컨트롤" |
| R0-R0 | C1+(전 C2/per-docker)을 한 흐름으로 묶음 | R0 final |
| R0-R0 | Diagnosis (stale image 검증, D4) drop — 호스트 직접 실행으로 가설 moot | R0 follow-up |
| R1 | ARM 지원 정의 = qemu-user ARM userspace (x86 호스트) | R1 |
| R2 | Kernel CTF tools defer (후속 목표, 연구 필요) | R2 |
| R3 | ARM 지원도 defer | R3 |
| R4 | Ship gate = minimal demo (worker spawn + 1 fixture binary) | R4 contrarian |
| R5 | Worker container base = 기존 fork Dockerfile 재활용 + 필요시 로컬 customize | R5 |
| R6 | Minimal demo 단계는 일반 MCP client로 wire-up 검증 → OmP-specific wiring은 후속 | R6 simplifier |

## Sequential sub-tasks (current pipeline)

> 각 sub-task = 1 commit (CLAUDE.md "Logical change 단위" 룰).
> 진행 중 발견되는 implementation detail은 sub-task 안에서 incremental 결정 + spec/decisions.md 보강.

- [ ] **#1 Setup — `feat/per-docker-refactor` 브랜치 + Dockerfile worker 모드 검토**
  - `git checkout -b feat/per-docker-refactor`
  - `docker build -t omp/pwno-mcp:dev .` 성공
  - `docker run --entrypoint /bin/bash --rm -it omp/pwno-mcp:dev gdb --version` 응답 (worker용 entrypoint override 가능 확인)
- [ ] **#2 Phase 1a wire-up 검증 — 호스트 startup + 일반 MCP client 연결**
  - `uv sync` + `uv run python -m pwnomcp` (or `--stdio`)
  - Claude Code 한 군데에서 pwno-mcp 연결 + tool 목록 노출 + read-only tool 1개 (예: `health` 또는 `list_processes`) 호출 성공
- [ ] **#3 `ContainerSession` + `DockerWorkerRegistry` skeleton 추가 (병행)**
  - `pwnomcp/state/registry.py` additive (기존 `DebugSession`/`DebugSessionRegistry` 제거 안 함)
  - `ContainerSession` 필드: `session_id`, `container_id`, `gdb_host`, `gdb_port`, `runtime_dir`, `state`, `ready`
  - 새 클래스 import + unit test 1개 (mock docker SDK) 통과
- [ ] **#4 Worker spawn / cleanup 핵심 path 구현**
  - `DockerWorkerRegistry.spawn_worker_container(session_id) -> ContainerSession`: `docker run --entrypoint sleep --rm -d -v $(pwd)/workspace:/workspace omp/pwno-mcp:dev infinity` (예시)
  - `close_worker(session_id)`: `docker stop` + `docker rm`
  - Health probe: `docker exec` 로 `gdb --version` 호출 성공 여부
  - Worker mode toggle (env var 예: `PWNO_WORKER_MODE=1`) 정의
- [ ] **#5 `pwno_create_debug_session` worker mode 분기 wire**
  - `pwnomcp/tools/debug.py` 분기 추가 (worker mode flag 확인 → `DockerWorkerRegistry` 경로)
  - `pwnomcp/tools/common.py` `resolve_debug_session` 분기 또는 새 helper
  - 기존 in-process gdb 경로 default 유지 (회귀 0, 1 sanity case 검증)
- [ ] **#6 Minimal demo (ship gate) — fixture ELF로 worker spawn → gdb-mi → cleanup**
  - fixture: `tests/fixtures/hello_x86_64` (없으면 `gcc -o hello hello.c` 로 만들고 commit)
  - `pwno_create_debug_session(binary="/workspace/hello_x86_64")` worker mode → container_id 응답
  - `docker exec` 로 gdb-mi 명령 1회 응답 (`-data-evaluate-expression "$pc"` 또는 `-exec-step`)
  - `pwno_close_debug_session` 호출 → container `docker stop` + `rm` 정상
  - **여기까지 통과 → ship gate. PR-ready commit.**
- [ ] **#7 Docs 후속 — decisions.md D5 추가 + CLAUDE.md 분기 정정 + prev-task 아카이브**
  - `.omc/decisions.md` D5: "host pwno-mcp + per-docker worker (분기 A) 채택, B 폐기"
  - `CLAUDE.md` "분기 (B) ... **추천.**" 라인 → 분기 A 채택으로 갱신
  - 본 current-task 내용을 `prev-task.md`에 아카이브, current-task 다음 phase로 비움 (또는 ARM/Kernel 후속 spec 진입 결정)

## Open implementation questions (sub-task 안에서 결정)

- pwncli inside worker vs host-side (sub-task #4-#5 진행 중 결정)
- gdb communication protocol — `docker exec` MI vs worker 안 REST (sub-task #4-#5)
- worker heartbeat / liveness probe (sub-task #4)
- session cleanup on worker crash / orphan reclaim (sub-task #4)
- Worker resource limit (memory/cpu cgroup) — minimal demo 이후

## Out of scope (이번 pipeline)

- ARM 지원 (deferred — 후속 spec, `feat/arm-support` 브랜치 예정)
- Kernel CTF tools (deferred — 사용자 연구 + 후속 spec, `feat/kernel-debug` 브랜치 예정)
- OmP-specific wiring (`OMP_PWNO_IMAGE` env var, OmP 측 endpoint 전환 로직) — 본 ship gate 후 별개 sub-task
- 새 minimal worker image 정의 (기존 Dockerfile 재활용)
- Docker-in-docker (분기 B 폐기)
- Multi-host / orchestrator (k8s, swarm)
- API 호환성 보증 (vanilla 36 tool 시그니처 1:1 유지) — 본 spec은 1개 tool path만 검증
- macOS / Windows 호스트 지원
- upstream PR (사용자 개인 fork)
- 다른 client(claude-code/cursor/codex-cli) 정식 호환 유지 — Phase 1a 검증 도구로만 사용

## Reference

- Spec: `.omc/specs/deep-interview-c1-host-mcp-docker-tool.md`
- Deep-interview state: `.omc/state/deep-interview-state.json`
- decisions.md: D1–D4 (D5 추가 예정 — sub-task #7)
- explore agent 보고: spec § Technical Context

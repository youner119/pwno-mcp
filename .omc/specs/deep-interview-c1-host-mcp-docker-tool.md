# Deep Interview Spec: C1 Host pwno-mcp + Docker-as-tool refactor

## Metadata
- Interview ID: `2152222a-0d46-4674-95e7-d3aa60ca3743`
- Rounds: 7 (Round 0 topology + R1–R6)
- Final Ambiguity Score: **17.0%** (threshold 20%)
- Type: brownfield (pwno-mcp youner119 fork, branch `main`)
- Generated: 2026-05-15
- Threshold: 0.20
- Initial Context Summarized: no (in-conversation context sufficient)
- Status: **PASSED** (≤ threshold)
- Challenge modes used: Contrarian (R4), Simplifier (R6)
- Spec path: `.omc/specs/deep-interview-c1-host-mcp-docker-tool.md`

## Clarity Breakdown
| Dimension | Score | Weight | Weighted |
|-----------|-------|--------|----------|
| Goal Clarity | 0.80 | 0.35 | 0.280 |
| Constraint Clarity | 0.85 | 0.25 | 0.2125 |
| Success Criteria | 0.90 | 0.25 | 0.225 |
| Context Clarity | 0.75 | 0.15 | 0.1125 |
| **Total Clarity** | | | **0.830** |
| **Ambiguity** | | | **0.170 (17.0%)** |

## Topology
| Component | Status | Description | Coverage / Deferral Note |
|-----------|--------|-------------|--------------------------|
| **C1: Host pwno-mcp + Docker-as-tool refactor** | active | pwno-mcp를 호스트 Python process로 실행. pwno-mcp가 host docker daemon 통해 session id ↔ worker container 매핑. (CLAUDE.md 분기 A) | 본 spec 전 영역 cover. 모든 acceptance criteria 이 component 대상 |
| C2: ARM 지원 | deferred | qemu-user ARM userspace target (x86 호스트에서 ARM ELF binary CTF) | 사용자 R3: "ARM도 후속 — 지금 pipeline에서 defer". `feat/arm-support` 후속 spec 필요 |
| C3: Kernel CTF tools | deferred | qemu-system + vmlinux first-class kernel debug tools | 사용자 R2: "연구 필요. kernel ctf는 후속 목표". `feat/kernel-debug` 후속 spec + 리서치 필요 |
| Diagnosis (stale image 검증, D4) | dropped | vanilla `:latest` image fresh pull 후 Round 2 timeout 재현 | 사용자 R0 follow-up: 호스트 직접 실행으로 전환 → 가설 자체가 moot. decisions.md D4 obsolete |

## Goal

pwno-mcp(이 fork)를 **호스트 Python process로 직접 실행**하도록 전환하고, MCP tool 호출이 들어오면 pwno-mcp가 **host docker daemon을 통해 debug session 단위로 worker container를 spawn / control / cleanup** 한다 (CLAUDE.md "분기 A: pwno-mcp가 호스트에서 동작 + docker CLI/SDK 사용" 채택, "분기 B docker-in-docker" 폐기). OmP 측 wiring은 본 spec 의 minimal demo ship gate에 포함 안 함 — 일반 MCP client (Claude Code / Codex 등) 으로 wire-up만 검증, OmP-specific endpoint 결정은 후속 sub-task.

**Sequential pipeline 첫 워크스트림. Sub-task 분해 후 `current-task.md`에 순차로 적재됨.**

## Constraints

- **Branch:** 모든 작업은 `feat/per-docker-refactor` (신규) 브랜치에서. main은 upstream 추적 유지.
- **Worker base image:** 기존 `Dockerfile` (이 fork) 빌드 결과 재활용 (`docker build -t omp/pwno-mcp:dev .`). 새 minimal worker image 별도 정의 안 함. 필요 시 fork에서 Dockerfile customize 후 로컬 rebuild.
- **Worker entrypoint:** 기존 image의 ENTRYPOINT (`python -m pwnomcp`)는 worker 모드에서 부적합 → `docker run --entrypoint ...` 로 override (e.g. `sleep infinity` + `docker exec` 패턴).
- **Architecture 분기:** A 확정 (host pwno-mcp + worker docker container들). B (docker-in-docker, sock 마운트) 폐기.
- **회귀 정책:** 기존 in-process gdb session 경로는 default로 *유지* (worker mode는 toggle/flag로 enable). 회귀 0이 first sub-task의 묵시적 acceptance.
- **OmP 호환성:** 본 spec scope 안에는 OmP wiring 변경 *없음*. OmP는 계속 vanilla 사용 가능. fork worker mode 검증은 일반 MCP client로.
- **호스트 OS:** Linux (사용자 환경 기준). macOS/Windows는 명시적 out of scope.
- **MCP transport:** stdio 또는 HTTP — 일반 MCP client가 받는 형태 그대로. 새 transport 도입 없음.

## Non-Goals

- OmP-specific wiring (`OMP_PWNO_IMAGE` env var, OmP 측 endpoint 전환 로직) — 후속 sub-task로 분리.
- ARM 지원, Kernel CTF tools — 별개 component (deferred, 위 Topology 참조).
- 새 worker base image 정의 — 기존 Dockerfile 재활용으로 충분.
- API 호환성 보증 (vanilla 36 tool 시그니처 1:1 유지) — minimal demo는 1개 tool path (`pwno_create_debug_session`) 만 검증. 나머지 tool migration은 후속 sub-task.
- Docker-in-docker (분기 B) — 폐기 결정.
- Multi-host / orchestrator (k8s, swarm) — 단일 호스트 가정.
- Worker resource limit / cleanup policy 정밀화 (idle timeout, orphan reclaim) — 후속.
- pwncli inside worker vs host-side 결정 — open question, sub-task 안에서 incremental.
- gdb communication protocol 결정 (MI over docker exec vs REST inside worker) — open question.

## Acceptance Criteria

**Phase 1a — MCP wire-up (사용자 R6 답: "일반적인 claude code, codex에 mcp 연결 확인. 그 후에 기능 테스트"):**
- [ ] `uv sync` 호스트에서 성공
- [ ] `uv run python -m pwnomcp` (또는 `--stdio`) 호스트 직접 실행 시 정상 startup
- [ ] 일반 MCP client (Claude Code 한 군데로 충분) 에서 pwno-mcp 연결 + tool 목록 노출 확인
- [ ] read-only tool 1개 (예: `health` 또는 `list_processes`) 호출 → 정상 응답

**Phase 1b — Worker spawn function demo:**
- [ ] `feat/per-docker-refactor` 브랜치 생성, `omp/pwno-mcp:dev` 로컬 build 성공
- [ ] worker mode toggle (env var 또는 config flag) 정의
- [ ] `pwno_create_debug_session` worker mode 분기에서 `omp/pwno-mcp:dev` 로 worker container spawn → container_id 응답
- [ ] worker 안에 `docker exec` 로 `gdb --version` 호출 성공 (health probe)
- [ ] 1 fixture ELF (`tests/fixtures/hello_x86_64` 등 간단한 binary) 에 대해 gdb-mi 명령 1회 (`-exec-step` 또는 `-data-evaluate-expression`) 응답
- [ ] `pwno_close_debug_session` 호출 → worker container `docker stop` + `docker rm` 정상 동작
- [ ] 기존 in-process gdb 경로 (worker mode off) 회귀 0 — 1개 vanilla 시나리오 sanity check

**Ship gate (commit 단위):**
- 위 Phase 1a + 1b 모두 통과 → minimal demo 완료, `feat/per-docker-refactor` 브랜치에서 PR-ready commit. 사용자가 main 머지 여부 결정.

## Assumptions Exposed & Resolved

| Assumption | Challenge | Resolution |
|------------|-----------|------------|
| pwno-mcp는 docker container 안에서 동작해야 한다 (Docker-first) | R0 follow-up: "stale image가 먼데?" → "로컬에서 돌고 docker는 디버깅을 위한 걸로" | pwno-mcp는 호스트 Python process로 동작. docker는 *worker* 역할. CLAUDE.md 분기 B "추천" 폐기. |
| 분기 B (docker-in-docker, sock 마운트)가 사용자 친화 → 추천 | R0 final: "1번과 2번 합쳐 한 흐름" + "로컬 mcp가 docker들 컨트롤" | 분기 A 채택. 사용자가 호스트 직접 실행 + worker 분리 선호 명시. |
| Stale image 가설을 fork 작업 전 검증해야 한다 (D4) | R0: 호스트 직접 실행으로 전환 → vanilla `:latest` 안 씀 | D4 obsolete (Diagnosis dropped). decisions.md 업데이트 필요. |
| ARM/Kernel CTF는 이번 sequential pipeline에 같이 들어간다 | R2: "kernel ctf는 후속 목표" / R3: "ARM도 후속" | 둘 다 deferred. C1만 active. spec/current-task scope 좁아짐. |
| Ship gate = OmP 실 challenge 통과 (full integration) | R4 Contrarian: 진짜 ship gate가 minimal일 수도 | Minimal demo (worker spawn + 1 fixture binary). 빠른 iteration + commit 단위 명확. |
| Worker container 별 새 minimal image 정의 필요 | R5: "현재 폴더에 dockerfile 있지 않아? ... 커스텀으로도 할 수 있" | 기존 Dockerfile 재활용 + 필요시 로컬 customize. 별도 image 정의 폐기. |
| OmP 연결 방식이 minimal demo ship gate에 포함됨 | R6 Simplifier: "일반적인 claude code, codex에 mcp 연결 확인. 그 후에 기능 테스트" | OmP wiring은 후속 sub-task. minimal demo는 일반 MCP client로 wire-up 검증. |

## Technical Context (brownfield codebase mapping)

`explore` agent 보고 (Round 6) 기반.

**현재 single-container multi-session 인프라:**
- `pwnomcp/state/registry.py:20-46` — `DebugSession` dataclass: `session_id`, `runtime_dir`, `gdb` (`GdbController` 인스턴스), `state` (`SessionState`), `tools` (`PwndbgTools`), `lock` (`threading.RLock`), `driver_pid`
- `pwnomcp/state/registry.py:48-144` — `DebugSessionRegistry`: process-wide singleton, `Dict[session_id → DebugSession]`, `_lock` (RLock), `default_session_id`
- `pwnomcp/services.py:20-31` — `AppServices`: registry singleton + `pwnpipe_sessions` (per-session Dict) + `pwnpipe_lock`
- `pwnomcp/services.py:33-62` — `create_services()`: instantiates registry, creates default session with `GdbController`, binds to lifespan
- `pwnomcp/tools/common.py:54-71` — `resolve_debug_session()`: registry lookup or create
- `pwnomcp/tools/common.py:92-100` — `run_session_action()`: `asyncio.to_thread(action inside session.lock)`
- `pwnomcp/tools/debug.py:19-28` — `create_debug_session` MCP tool entry
- `pwnomcp/tools/debug.py:42-57` — `close_debug_session`: pops pwnpipe + closes registry session
- `pwnomcp/tools/pwncli.py:22-130` — `pwncli` tool: `PwnPipe` subprocess per session
- `pwnomcp/tools/backends/gdb.py:19-32` — `GdbController.__init__`: `pwndbg --interpreter=mi3` subprocess
- `pwnomcp/pwnpipe.py:10-66` — `PwnPipe`: subprocess.Popen wrapper + reader/waiter daemon threads
- `pwnomcp/lifespan.py:9-25` — `create_lifespan`: yields `{"services": managed_services}`

**권고 추상화 매핑 (host-mode + per-docker-worker):**
- `DebugSession` → `ContainerSession` (registry.py): 제거 `gdb`/`lock`/`tools`/`driver_pid`, 추가 `container_id`/`gdb_host`/`gdb_port`/`ready`
- `DebugSessionRegistry` → `DockerWorkerRegistry` (registry.py): 유지 dict + lock + default, 추가 docker client 또는 subprocess CLI, image tag, startup timeout
- `PwndbgTools` → `RemoteGdbToolsProxy` (new file): docker exec or REST, retry/timeout 흡수
- `AppServices.pwnpipe_sessions` → per-container stdin/stdout (services.py): pwncli 위치 (host vs worker) 결정 후 매핑
- `run_session_action()` → `docker_exec_action()` (tools/common.py): docker exec 래핑, error catching/state sync 유지
- session cleanup → `docker stop` + `docker rm` + volume cleanup (debug.py `close_debug_session`)

**Refactor 위험 / 주의 영역:**
- `PwnPipe` daemon threads 누적 — 새 architecture에서 host-side 잔존 시 cleanup 정책 필요
- pwncli pipe close 로직 (`44349c7 fix(router): resolve session id when closing pwncli pipes`) 호환성
- event-loop blocking fix (`c678df1`) 가 새 docker exec 호출에도 적용되는지 확인
- worker readiness probe 실패 시 fallback (예: container 살아있지만 gdb 응답 안 함)

## Sequential Sub-task Plan

`current-task.md` 에 그대로 적재될 sub-task 순서. 각 sub-task = commit 단위 (사용자 working style: "Logical change 단위로 commit 묶기").

| # | Sub-task | Files (예상) | Acceptance |
|---|----------|--------------|------------|
| 1 | `feat/per-docker-refactor` 브랜치 생성 + 기존 Dockerfile worker 모드 검토 (entrypoint override 가능 여부 확인) | `Dockerfile` (read only) | 브랜치 생성 + `docker build -t omp/pwno-mcp:dev .` 성공 + `docker run --entrypoint /bin/bash --rm -it omp/pwno-mcp:dev gdb --version` 응답 |
| 2 | 호스트 startup 검증 (Phase 1a wire-up) — `uv run python -m pwnomcp` 호스트 직접 실행 + 일반 MCP client (Claude Code) 한 곳에서 tool 목록 노출 + read-only tool 1개 호출 성공 | (없음 — runtime 검증) | Phase 1a acceptance 통과 |
| 3 | `ContainerSession` dataclass + `DockerWorkerRegistry` skeleton 추가 (기존 `DebugSession`/`DebugSessionRegistry` 와 *병행*, 제거 안 함) | `pwnomcp/state/registry.py` (additive) | 새 클래스 import 가능, unit test 1개 (mock docker SDK) 통과 |
| 4 | Worker spawn / cleanup 핵심 path: `DockerWorkerRegistry.spawn_worker_container(session_id) → ContainerSession` + `close_worker(session_id)` 구현 | `pwnomcp/state/registry.py`, `pwnomcp/services.py` (worker mode toggle) | 단위로 worker container spawn → `docker exec gdb --version` 성공 → `docker stop`/`rm` 동작 |
| 5 | `pwno_create_debug_session` worker mode 분기 wire — 환경변수/flag로 toggle, worker mode 일 때 `DockerWorkerRegistry` 경로 사용 | `pwnomcp/tools/debug.py`, `pwnomcp/tools/common.py` | tool 호출 시 worker mode 분기 정상 동작 + 기존 in-process 경로 회귀 0 (1 sanity case) |
| 6 | Minimal demo: 1 fixture ELF (`tests/fixtures/hello_x86_64`) → worker spawn → `docker exec` 로 gdb-mi 명령 1회 응답 → cleanup | `tests/` 또는 manual demo, `pwnomcp/tools/backends/` (RemoteGdbToolsProxy stub 가능) | Phase 1b acceptance 통과 (ship gate) |
| 7 | `decisions.md` D5 추가 ("분기 A 채택 / B 폐기") + CLAUDE.md "분기 B 추천" 라인 갱신 + spec/sub-task 진행 결과 `prev-task.md` 로 아카이브 | `.omc/decisions.md`, `CLAUDE.md`, `.omc/state/prev-task.md`, `.omc/state/current-task.md` | docs 일관성 — 분기 결정이 spec / decisions / CLAUDE 모두에 반영 |

**예상 commit 7개. 1–2번이 setup, 3–5번이 abstraction, 6번이 ship gate, 7번이 docs 후속.**

각 sub-task 진행 중 발견되는 implementation detail (open questions: pwncli host vs worker, gdb 통신 protocol, heartbeat, cleanup) 은 *해당 sub-task 안에서 incremental 결정* + decisions.md / spec 보강.

## Ontology (Key Entities) — Round 6 final

| Entity | Type | Fields | Relationships |
|--------|------|--------|---------------|
| pwno-mcp (host process) | core | host Python process, MCP server | controls worker containers |
| worker container | core | docker container, per session | spawned by pwno-mcp, runs gdb/pwncli |
| session id | core | string identifier | maps 1:1 to worker container |
| ContainerSession | proposed core | session_id, container_id, gdb_host, gdb_port, ready | replaces DebugSession |
| DockerWorkerRegistry | proposed core | dict[session_id → ContainerSession], docker client | replaces DebugSessionRegistry |
| RemoteGdbToolsProxy | proposed core | gdb_host, gdb_port, container_id | replaces PwndbgTools |
| host docker daemon | external | unix socket / API | pwno-mcp → docker daemon → spawn worker |
| MCP wire-up | core | transport (stdio/HTTP), tool list, handshake | gating Phase 1a |
| fixture binary | supporting | ELF, e.g. hello_x86_64 | gating Phase 1b |
| ship gate | meta | Phase 1a + 1b 통과 | commit 단위 정의 |
| OmP plugin (consumer) | external | TypeScript opencode plugin | 본 spec scope 밖 wiring |
| Claude Code / Codex (MCP client) | external | 일반 MCP client | Phase 1a wire-up 검증 도구 |
| Dockerfile (이 fork) | reference | Ubuntu 24.04 + pwn 의존성 | worker base |
| qemu-user / qemu-system / vmlinux | external | binary tools | C2/C3 deferred component 영역 |

## Ontology Convergence

| Round | Entity Count | New | Changed | Stable | Stability Ratio |
|-------|--------------|-----|---------|--------|-----------------|
| 1 | 10 | 10 | - | - | N/A |
| 2 | 11 | 1 | 0 | 10 | 0.91 |
| 3 | 11 | 0 | 0 | 11 | 1.00 |
| 4 | 13 | 2 (ship gate, fixture binary) | 0 | 11 | 0.85 |
| 5 | 14 | 1 (worker spawn flow) | 0 | 13 | 0.93 |
| 6 | ~16 | 5 (registry, services, ContainerSession, DockerWorkerRegistry, MCP wire-up) | 1 (DebugSession → ContainerSession 매핑) | 13 | 0.875 |

수렴 양상: R3에서 1.0 도달했다가 R4 contrarian + R6 explore agent 결과로 새 entity 유입. 그러나 추가된 entity는 모두 *기존 abstraction의 매핑/대체*라 의미적 안정성은 유지.

## Interview Transcript

<details>
<summary>Full Q&A (R0 + R1–R6)</summary>

### Round 0 (topology gate)
**Q:** Top-level workstream topology가 맞아? (3 components: Diagnosis / Per-docker / Kernel CTF)
**A v1:** 둘 다 별개 workstream (ARM/Kernel separate)
**Follow-up clarify:** "stale image가 먼데?" → 설명 후 사용자 답: "기존의 docker를 버리고 이제 로컬 도커 빌드를 해야 하지 않을"
**Follow-up clarify 2:** "1번 mcp를 하기 위해서 도커를 하는 거야?" → 설명 후 사용자 답: "내가 원하는 건 로컬 상에서 돌아가고 도커는 디버깅 하기 위한 걸로 ... 로컬 mcp가 각 docker들을 컨트롤"
**Final A:** Component 4개 맞으나 1번+2번 합쳐 한 흐름. Diagnosis drop. → Topology v3 (3 active: C1 Host pwno-mcp+Docker-as-tool / C2 ARM / C3 Kernel CTF, Diagnosis dropped)
**Ambiguity:** N/A (Round 0 = topology only)

### Round 1
**Q:** ARM 지원이 구체적으로 무엇을 의미해?
**A:** qemu-user ARM userspace
**Ambiguity:** 71.5% (was 83.5%, ↓12%)
**Targeted:** C2 / Goal (was 0.20)

### Round 2
**Q:** Kernel CTF tools 자동화 범위는 어디까지?
**A:** 연구 필요. kernel ctf는 후속 목표일 뿐 지금 중요한 건 아
**Ambiguity:** 59% (was 71.5%, ↓12.5% — C3 deferred)
**Targeted:** C3 / Goal (was 0.40)
**Topology change:** C3 → deferred

### Round 3
**Q:** ARM 지원의 시점 위치는? (defer? 가까이? 평행?)
**A:** ARM도 후속 — 지금 pipeline에서 defer
**Ambiguity:** 59% (no change — weakest dim 그대로 C1)
**Targeted:** C2 (meta/topology)
**Topology change:** C2 → deferred. Active 1 (C1 only).

### Round 4 (Contrarian mode)
**Q:** C1 ship gate 기준은?
**A:** Minimal demo — worker spawn flow + 1 fixture binary
**Ambiguity:** 36.75% (was 59%, ↓22%)
**Targeted:** C1 / Criteria (was 0.20)

### Round 5
**Q:** Worker container base image 정책은?
**A:** "현재 폴더에 dockerfile 있지 않아? ... 커스텀으로도 할 수 있" → 기존 Dockerfile 재활용 + 필요시 로컬 customize
**Ambiguity:** 27.5% (was 36.75%, ↓9.25%)
**Targeted:** C1 / Constraints (was 0.40)

### Round 6 (Simplifier mode + explore agent)
**Q:** Minimal demo 단계에 진짜 필요한 constraint는?
**A:** 일반적인 claude code, codex에 mcp 연결 확인. 그 후에 기능 테스트
**Explore agent:** registry.py / services.py / tools/common.py 매핑 + abstraction 권고 (ContainerSession / DockerWorkerRegistry / RemoteGdbToolsProxy)
**Ambiguity:** **17.0%** (was 27.5%, ↓10.5%) ✅ threshold 도달
**Targeted:** C1 / Constraints (0.70 → 0.85, Context 0.50 → 0.75)

</details>

# pwno-mcp fork — Architectural Decisions

> docs / specs / git history에 못 들어간 reasoning 보존.

---

## D1: Fork over patches (2026-05-15)

**결정:** `youner119/pwno-mcp` 별개 fork repo로 분리. upstream PR 의도 없음.

**근거:**
- OmP 로드맵에 kernel CTF 포함 → 상당량의 OmP-specific tool 추가 예정 (`pwno_kernel_*` 시리즈, KASLR 자동화, guest-host file transfer 등). 이 작업은 upstream의 generic-MCP 디자인 의도와 어긋날 가능성 큼.
- Per-docker architecture refactor 또한 upstream 단일 컨테이너 디자인과 큰 충돌. PR negotiation 비용이 직접 fork 유지 비용보다 클 가능성.
- 사용자가 fork 단독 유지 의향 명시 ("내가 쓸거야").

**Trade-off 수용:**
- Maintenance: upstream 변경마다 `git fetch upstream && git merge` 또는 rebase. 활발한 프로젝트라 주기적 sync 비용 있음.
- 다른 client(claude-code, cursor 등)에선 vanilla pwno-mcp 권장. 이 fork는 OmP 외 호환성 무보장.
- 사용자 개인 머신에 한정. 다른 머신 부트스트랩 시 `gh repo fork` + clone 동일 반복 필요.

---

## D2: 컨텍스트 분리 — pwno-mcp 측에 별도 .omc + CLAUDE.md (2026-05-15)

**결정:** 이 fork에 자체 `CLAUDE.md` + `.omc/` 둠. OmP repo에는 짧은 참조만.

**근거:**
- pwno-mcp는 Python + Docker + gdb stack. OmP는 TypeScript + opencode plugin. 두 stack을 한 CLAUDE.md에 섞으면 컨텍스트 무거움.
- 각 Claude session이 자기 영역에만 집중하면 더 빠른 부팅 + 정확한 작업 안내.
- Task tracking 분리: OmP는 plugin/agent 작업, 이 fork는 architecture/tool 작업. current-task.md 둘이 각자 관리.
- Spec docs locality: `.omc/specs/per-docker-architecture.md`, `.omc/specs/kernel-ctf-tools.md` 등은 구현될 코드 옆에 있는 게 자연.

**Trade-off:**
- 두 CLAUDE.md 동기화: 작업 방향(operating rules, working style)이 양쪽에 중복. 다행히 user working style은 거의 동일해서 복붙해도 됨.
- Cross-project 결정 (예: OmP의 `OMP_PWNO_IMAGE` env var 도입): 양쪽 current-task.md에 모두 언급 필요. 현 시점은 OmP CLAUDE.md에 짧게만 기록.

---

## D3: .omc/ + CLAUDE.md를 fork에 commit (2026-05-15)

**결정:** 이 파일들은 `.gitignore`에 추가하지 않고 `origin/main`에 commit.

**근거:**
- upstream PR 의도 없음 (D1) → upstream 영역(`pwnomcp/`, `docs/` 등)을 깨끗하게 유지할 필요 없음.
- 다른 머신에서 사용자가 fork pull 시 자동으로 컨텍스트 따라옴 (CLAUDE.md + task state).
- upstream `pwno-io/pwno-mcp`에는 `.omc/` 디렉토리가 존재하지 않으므로 upstream 머지 시 충돌 없음.

**Trade-off:**
- 우리 fork repo가 OmP-specific 표시 명확히 보임 (공개 fork지만 사적 용도라 문제 안 됨).
- 만약 향후 PR 의도 생기면 PR 브랜치 따로 만들어서 `.omc/` 제외하고 cherry-pick.

---

## D4: Stale image 가설 검증을 fork 작업 진행 전에 수행 (2026-05-15)

**결정 (잠정):** Per-docker refactor 본격 시작 전에 vanilla `:latest` 이미지 fresh pull + OmP에서 Round 2 재현 시도. 결과 따라 fork 작업 우선순위 조정.

**근거:**
- upstream main에 이미 multi-session 인프라 + event-loop fix 들어가 있음. 우리가 본 Round 2 timeout이 stale local image 때문일 수 있음.
- 검증 비용 10분 수준. 결과가 어느 쪽이든 정보 가치 큼.
- timeout 사라지면 → per-docker refactor는 kernel CTF 위한 architectural cleanup으로 격하 (시급도 낮아짐, 그러나 가치는 여전)
- timeout 여전 → vanilla에도 진짜 buggy → fork에서 root-cause fix 필수

**Out of scope (현 시점):**
- 검증 결과 따른 후속 액션 (per-docker refactor 우선순위 조정 / 다른 root-cause 진단).

> **D4 obsolete (2026-05-15):** D5 결정 (분기 A 채택)으로 vanilla `:latest` image 사용 자체가 폐기 → stale image 가설이 moot. Diagnosis 액션 dropped. 본문은 reasoning trail로 보존.

---

## D5: 분기 A 채택 (host pwno-mcp + per-docker worker), 분기 B 폐기 (2026-05-15)

**결정:** `feat/per-docker-refactor` 의 architectural 분기는 **A** (pwno-mcp가 호스트 Python process로 동작 + docker CLI/SDK 또는 subprocess로 worker container 매핑). **B** (docker-in-docker, `docker.sock` 마운트 sibling spawn) **폐기**.

**근거 (deep-interview 결과 — `.omc/specs/deep-interview-c1-host-mcp-docker-tool.md` § Assumptions Exposed & Resolved 참조):**
- 사용자 명시 의도 (Round 0 follow-up): "내가 원하는 건 로컬 상에서 돌아가고 도커는 디버깅 하기 위한 걸로 ... 로컬 mcp가 각 docker들을 컨트롤".
- 분기 A는 docker-in-docker 복잡도 / 권한 처리 복잡도 0. host pwno-mcp 직접 실행 → dev iteration 빠름 (`uv run python -m pwnomcp`).
- pwno-mcp host 의존성은 사용자 환경에 이미 갖춰짐 (uv, gdb, pwndbg, qemu-user 등). 사용자 setup 비용 0.
- D2 추천 분기 B는 `pwno-mcp:dev` image 안에서 sibling spawn 패턴이었으나, 사용자가 "로컬에서 돈다"고 명시 → 분기 A로 pivot.

**D4 obsolete 효과:**
- D4 (stale image 가설 검증)는 *vanilla `:latest` image* 사용을 전제. 이번 결정으로 vanilla image 사용 자체가 폐기 → 가설 moot. 검증 액션 dropped.

**후속:**
- C1 ship gate (minimal demo, spec 참조) 통과 → 분기 A 검증 완료.
- 분기 A 변경 폭은 spec § Technical Context 의 추상화 매핑 참조 (`DebugSession → ContainerSession`, `DebugSessionRegistry → DockerWorkerRegistry`, `PwndbgTools → RemoteGdbToolsProxy`, `run_session_action → docker_exec_action`).
- Sub-task pipeline (7개) → `.omc/state/current-task.md`.

**Trade-off (수용):**
- 사용자가 다른 머신에서 fork 사용 시 호스트 의존성 install 필요 (vanilla docker pull로 해결되지 않음). 사용자 환경 한정 fork이므로 acceptable.
- pwno-mcp가 host docker daemon 사용 → 권한 (docker group 또는 root) 필요. Linux 한정 (macOS/Windows out of scope).
- ARM (C2) / Kernel CTF (C3) component는 **deferred** — 별개 후속 spec/branch.

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

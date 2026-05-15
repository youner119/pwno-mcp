# pwno-mcp (youner119 fork) — OmP exploit observation backbone

**OmP-specific fork** of [`pwno-io/pwno-mcp`](https://github.com/pwno-io/pwno-mcp). upstream PR 의도 없음 — 사용자 개인 fork로 유지. 소비처는 OmP plugin (`/mnt/D/Hack/oh-my-pwn`).

## Read these first when starting a session

1. **`.omc/state/current-task.md`** — 진행 중 sub-task pipeline (현재 C1 7-sub-task) + ship gate
2. **`.omc/specs/deep-interview-c1-host-mcp-docker-tool.md`** — 현재 active workstream spec (분기 A 채택, ambiguity 17%, ship gate = minimal demo)
3. **`.omc/decisions.md`** — docs / spec / git history에 못 들어간 architectural reasoning (D1–D5)
4. **`AGENTS.md`** (upstream 그대로) — 코드 컨벤션, 인사이드 가이드
5. **`docs/architecture.mdx`** (upstream) — pwno-mcp 런타임 구조 (FastMCP / session registry / 백엔드)
6. **`pwnomcp/tools/`** — 36개 MCP tool 정의 (debug.py / inspect.py / processes.py / pwncli.py / python_env.py / repos.py / retdec.py)
7. **OmP side context (소비자):** `/mnt/D/Hack/oh-my-pwn/CLAUDE.md`

## Fork 책임 범위

- **upstream `pwno-io/pwno-mcp`:** vanilla 추적용. `git fetch upstream` 으로 동기화. main 브랜치는 항상 upstream과 가깝게 유지.
- **이 fork `youner119/pwno-mcp` (origin):** OmP-specific customization. main에 머지하기 전 별개 브랜치에서 작업.
- **소비처:** OmP plugin이 사용하는 docker image. 향후 OmP의 `OMP_PWNO_IMAGE` env var로 vanilla `ghcr.io/pwno-io/pwno-mcp:latest` ↔ local fork image (`omp/pwno-mcp:dev` 등) 전환 가능하게 만들 예정. 현 시점은 vanilla만 사용.

## Branches

- `main` — upstream 추적 + `.omc/` docs (spec / current-task / decisions / CLAUDE.md). 우리 코드 변경분은 직접 머지 전까지 들어가지 않음.
- `feat/per-docker-refactor` (sub-task #1에서 생성) — host pwno-mcp + per-docker worker (분기 A). C1 spec 진행 중. Sub-task pipeline은 `.omc/state/current-task.md`.
- `feat/arm-support` (예정, deferred) — qemu-user ARM userspace target (x86 호스트에서 ARM ELF 실행).
- `feat/kernel-debug` (예정, deferred) — qemu-system + vmlinux first-class kernel CTF tools. 사용자 추가 리서치 후 spec.

각 큰 변경은 자기 브랜치에서 isolation. 검증 후 main으로 머지.

## 개발 방향

> Active workstream은 항상 1개 (`current-task.md` 참조). 나머지는 deferred (별개 spec/branch 예정).

### 1. C1 — Host pwno-mcp + Docker-as-tool refactor (active, spec 완료)

**Status:** Deep-interview 종결 (ambiguity 17%, threshold 20% 도달). Spec: `.omc/specs/deep-interview-c1-host-mcp-docker-tool.md`. Sub-task pipeline 7개 적재됨 → `.omc/state/current-task.md`.

**결정 (분기 A 채택, B 폐기):**
- pwno-mcp는 **호스트 Python process**로 직접 실행 (`uv run python -m pwnomcp` 또는 `--stdio`). docker container 안 아님.
- pwno-mcp가 host docker daemon 통해 **각 debug session = 별개 worker docker container** spawn / control.
- Worker base = 기존 fork `Dockerfile` (`docker build -t omp/pwno-mcp:dev .`) 재활용. 새 minimal worker image 정의 안 함. 필요 시 fork에서 Dockerfile customize 후 로컬 rebuild.
- Diagnosis (D4 stale image 검증) **dropped** — 호스트 직접 실행으로 가설 moot.

**Ship gate (minimal demo):**
- Phase 1a: 일반 MCP client (Claude Code / Codex 등) → 호스트 pwno-mcp 연결 + tool 목록 + read-only tool 1개 응답
- Phase 1b: `feat/per-docker-refactor` 브랜치에서 worker container spawn → 1 fixture ELF로 `docker exec gdb-mi` 1회 응답 → cleanup

**Open implementation questions** (sub-task 안에서 incremental 결정):
- pwncli inside worker vs host-side
- gdb 통신 protocol (MI over `docker exec` vs REST inside worker)
- worker heartbeat / liveness probe
- session cleanup on worker crash / orphan reclaim

**OmP wiring (`OMP_PWNO_IMAGE` 등) 은 본 spec scope 밖 — 후속 sub-task.**

### 2. C2 — ARM 지원 (deferred, qemu-user ARM userspace)

사용자 deep-interview R3: "ARM도 후속 — 지금 pipeline에서 defer". 정의: x86 호스트에서 ARM ELF userspace binary CTF (qemu-user ARM target). Branch: `feat/arm-support` (예정).

C3 kernel CTF (아래)와는 분리된 별개 workstream. C1 ship gate 통과 후 spec 작성 결정.

### 3. C3 — Kernel CTF tools (deferred, 사용자 추가 리서치 필요)

사용자 deep-interview R2: "연구 필요. kernel ctf는 후속 목표일 뿐 지금 중요한 건 아". qemu-system + vmlinux first-class. Branch: `feat/kernel-debug` (예정).

후보 tool (deep-interview 시점 추정, 사용자 리서치로 우선순위 재결정 예정):
- `pwno_kernel_debug_create(id, vmlinux, initramfs, ...)` — qemu-system 자동 spawn + gdb stub
- `pwno_kernel_walk_task_list(id)` — task_struct 순회
- `pwno_kernel_dump_slab(id, cache_name)` — slub allocator 상태
- KASLR slide 자동 계산 (vmlinux symbol vs runtime addr)
- guest-host file 전달 helper (stager upload, exploit payload)

C1 / C2 마무리 후 사용자 리서치 → spec 작성 → 진행.

## Operating rules (OmP working style 동일)

- **Pre-action approval for non-trivial work.** 새 파일 추가 / 다중 파일 리팩터 / 설계 결정 / 외부 자료 광범위 조사 등 비자명한 작업은 계획·옵션 제시 → 사용자 승인 후 실행. 단순 read / grep / typecheck / test / build 같은 검증 동작은 승인 불필요. 모호한 명령엔 옵션 (a)/(b)/(c) 제시.
- **Commits 자율 처리.** typecheck + pytest + (필요 시) docker build 모두 통과한 시점이면 사용자에게 다시 묻지 말고 자율 commit. message는 기존 upstream 스타일 따르고 `Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>` footer 포함.
- **Tool delegation soft warning은 무시.** "Recommended: Delegate to executor agent instead" 같은 hook 경고는 soft warning이라 직접 Bash/Edit/Write 진행. 사용자가 명시적으로 delegate를 원하지 않는 한.
- **Logical change 단위로 commit 묶기.** 한 문제 = 한 commit. 별개 concern은 별개 commit. Sub-task = commit 단위 (current-task.md 항목당 1 commit 원칙).
- **메모리 / 작업 상태 분리 규칙 준수.** `.omc/state/current-task.md`에 없던 작업을 했다면 `.omc/state/prev-task.md`에 추가, current는 사용자 명시 전엔 손대지 않음.
- **Spec 변경은 deep-interview 결과 반영시만.** Sub-task 진행 중 발견된 implementation detail은 spec § Open Questions / decisions.md 로 흡수. spec 본문은 deep-interview 재실행 시에만 갱신.

## Build / test (pwno-mcp 전용)

```bash
cd ~/Tools/pwno-mcp

# 의존성 (pyproject.toml + uv.lock 기반)
uv sync

# 테스트
uv run pytest tests/                    # 전체
uv run pytest tests/test_attach_endpoint.py -v  # 특정

# Lint / type (upstream CI에 맞춰)
uv run mypy pwnomcp/
uv run ruff check pwnomcp/

# Docker image 빌드 (분기 A 의 worker용)
docker build -t omp/pwno-mcp:dev .

# 호스트 직접 실행 (분기 A — pwno-mcp 자체)
uv run python -m pwnomcp                # HTTP server :5500
uv run python -m pwnomcp --stdio        # stdio transport
```

**OmP wiring (현재는 본 spec scope 밖, 후속):**
- vanilla 사용: 그대로 `ghcr.io/pwno-io/pwno-mcp:latest` 사용. fork 영향 0.
- fork worker mode 검증: 일반 MCP client (Claude Code 등) 로 호스트 pwno-mcp 직접 연결.
- 향후 OmP env var (`OMP_PWNO_IMAGE` 또는 endpoint toggle) 도입은 별개 sub-task.

## Upstream 동기화

```bash
git fetch upstream
git checkout main
git merge upstream/main                 # vanilla 추적
# 또는 rebase 우리 feat 브랜치를:
git checkout feat/per-docker-refactor
git rebase upstream/main
```

upstream에 의미 있는 fix가 들어가면 `git log upstream/main --oneline -10` 으로 확인 + 우리 main에 머지.

## 사용자 측 작업 방식 (참고)

- 한국어 기본. Technical terms (gdb, pwndbg, libc, ptrace, KASLR 등) 영문 유지.
- pwn / kernel 깊이 이해. 기초 설명 생략.
- Input contract: Python (uv) + Docker (docker buildx).

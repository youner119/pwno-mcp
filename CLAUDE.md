# pwno-mcp (youner119 fork) — OmP exploit observation backbone

**OmP-specific fork** of [`pwno-io/pwno-mcp`](https://github.com/pwno-io/pwno-mcp). upstream PR 의도 없음 — 사용자 개인 fork로 유지. 소비처는 OmP plugin (`/mnt/D/Hack/oh-my-pwn`).

## Read these first when starting a session

1. **`.omc/state/current-task.md`** — 진행 중 작업 + pending decisions
2. **`AGENTS.md`** (upstream 그대로) — 코드 컨벤션, 인사이드 가이드
3. **`docs/architecture.mdx`** (upstream) — pwno-mcp 런타임 구조 (FastMCP / session registry / 백엔드)
4. **`pwnomcp/tools/`** — 36개 MCP tool 정의 (debug.py / inspect.py / processes.py / pwncli.py / python_env.py / repos.py / retdec.py)
5. **`.omc/decisions.md`** — docs / spec / git history에 못 들어간 architectural reasoning
6. **OmP side context (소비자):** `/mnt/D/Hack/oh-my-pwn/CLAUDE.md`

## Fork 책임 범위

- **upstream `pwno-io/pwno-mcp`:** vanilla 추적용. `git fetch upstream` 으로 동기화. main 브랜치는 항상 upstream과 가깝게 유지.
- **이 fork `youner119/pwno-mcp` (origin):** OmP-specific customization. main에 머지하기 전 별개 브랜치에서 작업.
- **소비처:** OmP plugin이 사용하는 docker image. 향후 OmP의 `OMP_PWNO_IMAGE` env var로 vanilla `ghcr.io/pwno-io/pwno-mcp:latest` ↔ local fork image (`omp/pwno-mcp:dev` 등) 전환 가능하게 만들 예정. 현 시점은 vanilla만 사용.

## Branches

- `main` — upstream 추적. 우리 변경분은 직접 머지 전까지 들어가지 않음.
- `feat/per-docker-refactor` (계획) — single-container + multi-session → per-id worker container architecture refactor.
- `feat/kernel-debug` (계획) — qemu-system / vmlinux first-class kernel CTF tools.

각 큰 변경은 자기 브랜치에서 isolation. 검증 후 main으로 머지.

## 개발 방향 (planned, spec 작성 전)

### 1. Per-docker architecture refactor
**현재 (vanilla):** 단일 컨테이너 안에서 `session_id`로 gdb subprocess 다중화. parallel session 누적 시 state 누수 가능성 (OmP Round 2 `create_debug_session` 60s timeout 관찰).

**변경:** 각 debug request마다 별개 worker container spawn. Docker 관리도 MCP tool로 노출.
- `pwno_debug_create(id, ...)` → `docker run` worker container → returns id
- `pwno_debug_set_file(id, ...)` → 해당 worker로 routing
- `pwno_debug_close(id)` → `docker stop` worker

**구현 분기:**
- (A) pwno-mcp가 호스트에서 동작 + docker CLI/SDK 사용 — 사용자 setup 변경됨
- (B) pwno-mcp가 docker.sock 마운트한 컨테이너로 동작 + sibling container spawn — 사용자 친화. **추천.**

자세한 설계는 `.omc/specs/per-docker-architecture.md` (작성 예정).

### 2. Kernel CTF tools (first-class)
사용자 로드맵에 커널 CTF 포함. vanilla는 `qemu-user`만 자동화, `qemu-system` + vmlinux 디버깅은 raw `execute` escape hatch로만 가능.

**추가 예정 tool 후보:**
- `pwno_kernel_debug_create(id, vmlinux, initramfs, ...)` — qemu-system 자동 spawn + gdb stub 연결
- `pwno_kernel_walk_task_list(id)` — task_struct 순회
- `pwno_kernel_dump_slab(id, cache_name)` — slub allocator 상태
- KASLR slide 자동 계산 (vmlinux symbol vs runtime addr)
- guest-host file 전달 helper (stager upload, exploit payload)

자세한 설계는 `.omc/specs/kernel-ctf-tools.md` (작성 예정).

## Operating rules (OmP working style 동일)

- **Pre-action approval for non-trivial work.** 새 파일 추가 / 다중 파일 리팩터 / 설계 결정 / 외부 자료 광범위 조사 등 비자명한 작업은 계획·옵션 제시 → 사용자 승인 후 실행. 단순 read / grep / typecheck / test / build 같은 검증 동작은 승인 불필요. 모호한 명령엔 옵션 (a)/(b)/(c) 제시.
- **Commits 자율 처리.** typecheck + pytest + (필요 시) docker build 모두 통과한 시점이면 사용자에게 다시 묻지 말고 자율 commit. message는 기존 upstream 스타일 따르고 `Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>` footer 포함.
- **Tool delegation soft warning은 무시.** "Recommended: Delegate to executor agent instead" 같은 hook 경고는 soft warning이라 직접 Bash/Edit/Write 진행. 사용자가 명시적으로 delegate를 원하지 않는 한.
- **Logical change 단위로 commit 묶기.** 한 문제 = 한 commit. 별개 concern은 별개 commit.
- **메모리 / 작업 상태 분리 규칙 준수.** `.omc/state/current-task.md`에 없던 작업을 했다면 `.omc/state/prev-task.md`에 추가, current는 사용자 명시 전엔 손대지 않음.

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

# Docker image 빌드 (로컬 사용)
docker build -t omp/pwno-mcp:dev .

# 로컬 실행 (개발 — Docker 없이)
uv run pwno-mcp serve                   # FastMCP HTTP server on :5500
```

**OmP에서 fork image 쓰려면 (예정 인터페이스):**
```bash
docker build -t omp/pwno-mcp:dev ~/Tools/pwno-mcp/
OMP_PWNO_IMAGE=omp/pwno-mcp:dev omp   # OmP가 env var 보고 image 선택 (env var 도입 후)
```

env var 구현 전까지는 vanilla 이미지를 우리 fork tag로 retag해서 사용 가능:
```bash
docker pull ghcr.io/pwno-io/pwno-mcp:latest
docker tag ghcr.io/pwno-io/pwno-mcp:latest omp/pwno-mcp:dev
```

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

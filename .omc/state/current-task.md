# pwno-mcp fork — Current Task State

> 이 fork는 OmP가 소비. 변경의 큰 그림은 `CLAUDE.md` 참조.
> 다음 액션이 명확할 때만 여기 적음. 완료는 `prev-task.md`로 이동.

**Last updated:** 2026-05-15
**Branch:** `main` (vanilla 추적 중, 우리 commits 0)

## Phase / 컨텍스트

OmP 측 challenge1 실측에서 Round 2 `pwno_create_debug_session` 60s timeout 관찰. parallel session 누적 state 가능성. 진단 + 처치 방향 결정 단계.

**upstream main에 이미 다음이 있다:**
- `cd1d843 feat: add session-scoped debug runtime and path normalization` — DebugSession dataclass + DebugSessionRegistry + per-session lock
- `44349c7 fix(router): resolve session id when closing pwncli pipes`
- `c678df1 fix(router): avoid event-loop blocking and tighten attach session mapping`
- `2fb9e45 feat!(mcp): require explicit session_id and drop PID-based routing`
- `77cfbdb refactor(fastmcp-v3)` (PR #18)

즉 multi-session 인프라는 vanilla main에 이미 있음. 우리가 본 Round 2 timeout이 stale local image 때문일 가능성 큼.

## Pending decisions (사용자 결정 대기)

1. **Stale image 가설 검증 먼저?** (cheap, 10분)
   - `docker pull ghcr.io/pwno-io/pwno-mcp:latest` + OmP에서 작은 challenge로 Round 2 재현
   - timeout 사라지면 → 이 fork는 kernel CTF 전용. per-docker refactor 시급도 낮아짐
   - timeout 여전 → vanilla에도 진짜 architecture 이슈. fork에서 root-cause fix 필요

2. **Per-docker refactor 진행 여부?**
   - cost 분석 결과 30s/challenge 오버헤드 (~0.4% wall-clock) — acceptable
   - kernel CTF에 자연스럽게 확장 (mode parameter)
   - 작업량 6–9일 (구현 분기 B 기준)
   - Stale image 가설이 맞으면 시급도 낮아짐

3. **Architecture 분기: A (host process) vs B (docker-in-docker)?**
   - 추천 B (사용자 친화)
   - docs/ 추가 변경 동반

4. **Kernel CTF spec 작성 시점?**
   - Per-docker refactor 끝낸 뒤가 자연 (mode parameter 통일된 후)
   - 또는 spec만 먼저 작성 (구현 보류) — 설계 안정성 확보

## Next actions (결정 후)

(아래는 결정 시 spec/플랜으로 옮길 후보들)

- A. Stale image 검증 (10분): `docker pull` → omp 재실행 → Round 2 재현
- B. `.omc/specs/per-docker-architecture.md` 작성 (1–2일, design only)
- C. `.omc/specs/kernel-ctf-tools.md` 작성 (1–2일, design only)
- D. Per-docker refactor 구현 (`feat/per-docker-refactor` 브랜치, 6–9일)
- E. Kernel CTF tools 구현 (`feat/kernel-debug` 브랜치, 큰 작업)

## Out of scope

- upstream PR (사용자 개인 fork)
- 다른 client(claude-code/cursor/codex-cli) 호환 유지 (소비처는 OmP 한 곳)

# AGENTS.md — rtabmap

Project-specific rules for agents working on `src/rtabmap`.

---

## Project Identity

This workspace customizes RTAB-Map for **semantic zone-based keyframe load/unload behavior**.
See `PROJECT_DESCRIBE.md` (workspace root) for full project context.

- Primary code target: `src/rtabmap`
- `rtabmap_ros` is off-limits unless explicitly requested
- Main customized runtime: `src/rtabmap/corelib/src/Rtabmap.cpp`

---

## Project Goal

Replace RTAB-Map's default working-memory behavior with semantic-zone-driven keyframe management:

- Load signatures relevant to the robot's current semantic area
- Unload older zones under memory pressure
- Reduce unnecessary load/unload churn
- Preserve useful map content for revisits and overlapping regions

---

## Key Files

| File | Contains |
|------|----------|
| `corelib/src/Rtabmap.cpp` | Zone init, classification, retrieval target, pre-retrieval validation, retirement/unload |
| `corelib/include/rtabmap/core/Rtabmap.h` | Runtime state for the semantic-zone flow |
| `data/zone_signatures.json` | Zone definitions, bounds, bootstrap zone (`initial_zone`) |

---

## Custom Semantics

- Zone-to-signature mappings: `data/zone_signatures.json`
- `initial_zone` selects the bootstrap active zone
- Overlapping regions are intentional — multiple zones can activate simultaneously
- Missing optimized pose or out-of-bounds pose → fallback to current active-zone set
- Memory validation uses final retrieval target and actual missing IDs to load

---

## Editing Rules

- Modify only `src/rtabmap` — never touch `rtabmap_ros`
- Prefer small, reviewable changes
- Preserve unrelated user edits
- No builds unless explicitly requested
- Pause before destructive actions

---

## Build / Verify

Build only when explicitly requested. Run from workspace root:

```bash
export MAKEFLAGS="-j6"
colcon build --symlink-install \
  --cmake-args -DCMAKE_PREFIX_PATH="/opt/ros/humble" \
  --packages-select rtabmap \
  --allow-overriding rtabmap
```

---

## Environment

- Hospital environment — zone bounds and evaluation flow are environment-specific
- Do not generalize zone layout unless the task explicitly aims to do so

---

## Review Criteria

When reviewing a completed change, check:

1. Does the change fit the semantic-zone design?
2. Does it preserve or improve WM/LTM transition correctness and memory-threshold handling?
3. Is it safe across overlap, revisit, fallback, and lifecycle-reset cases?
4. Did it stay within intended scope without avoidable side effects?
5. Is a follow-up TODO needed to prevent context rot or correctness drift?

### Review Non-Goals

- Style-only feedback without behavioral value
- Broad refactor requests unless they directly reduce correctness risk
- Assuming upstream RTAB-Map behavior is preferable to this repo's custom zone policy

---

## Logging

Activity log: `CODEX_ACTIVITY.md` (workspace root).

Each entry:

```
### [YYYY-MM-DD HH:MM KST]
**Request** — <task or TODO item>
**Summary** — <1~2 lines>
**Agents Used** — <list>
**Files Modified** — <paths>
**Commit** — <message or `none`>
**Outcome** — <result summary>
**Notes** (optional) — <context, root cause, caveats>
```

---

## Failure Prevention

- Avoid editing the same file 3+ times in a row — escalate instead
- Avoid running builds unless the user explicitly asks
- Do not modify `rtabmap_ros` unless explicitly instructed

# todo.md — rtabmap

## Backlog
- [x] Upstream 0.23.5 포팅 빌드/동작 검증 — 2026-04-21 빌드 성공 + 실행 확인 완료 (Session 2/3 경유)
- [x] Immunized set 1회 구축 + 차감 방식 (분석 리포트 7.2) — Session 1에서 TRANSFER 재설계 시 함께 적용됨 (Rtabmap.cpp:4706-4749). Session 5 정리
- [ ] TRANSFER `_zoneHistory.size() > 1` 가드 재검토 — single-zone 상태에서도 WM threshold 초과 시 cleanup 허용 여부 — target: corelib/src/Rtabmap.cpp:4722 부근

## Architecture Decisions
- 2026-04-14: Semantic zone 기반 signature 관리 개념 자체는 유지 — TPS 저하는 구현 오류에 기인하며, 개념을 변경할 이유 없음
- 2026-04-21: Upstream fork 유지 대신 upstream 0.23.5로 재이식 전략 채택 — upstream 최신 기능(loop closure 개선, intermediate nodes with MM, LIO-SAM 등) 활용 목적. merge 커밋 `4da0bbf5`에서 포팅 완료
- 2026-04-21: **메인 브랜치 승격** — `segment-on-0.23.5` → `segment`로 리네임하고, 기존 23.4 기반 `segment`는 `segment-0.23.4`로 보존(백업). 이후 개발/jetson 배포는 `segment`(0.23.5 기반)를 기준으로 한다. 원격 `origin/segment`는 여전히 23.4를 가리키므로 push 시 `origin/segment-0.23.4` 보존 후 `origin/segment` 갱신 필요(별도 사용자 확인).
- 2026-04-21: **ROS2/Qt5 빌드는 anaconda env 오염된 쉘에서 수행하지 않는다** — 쉘 PATH에 `anaconda3/bin`이 들어있으면 CMake가 anaconda의 Qt5/libcurl/libtiff를 system보다 먼저 발견해서 moc silent fail 및 링커 심볼 미해결 유발. 빌드 전 `echo $PATH | grep anaconda`가 비어있어야 함. (`.bashrc`에서 `export PATH=~/anaconda3/bin:~/anaconda3/condabin:$PATH` 수동 라인 제거 + `conda config --set auto_activate_base false`로 해결됨)
- 2026-04-21: **upstream 머지 후 첫 빌드 전 `git ls-files`와 upstream 간 파일 크기 대조로 소실 파일 전수 검사**를 권장 — `LoopClosureViewer.cpp`가 LF 정규화/머지 과정에서 0바이트로 소실된 사례. bash 스니펫은 Session 3 근인 주석 참고.
- 2026-04-27: **분석 리포트 7.3(Forget 일괄 처리) 보류** — 7.3의 본래 동기는 "forget 루프가 retrieval 경로에 직렬 누적"이었으나, 이는 Session 1의 7.4(forget을 retrieval 이후로 이동)으로 해소됨. 현재 `Rtabmap.cpp:4722` 잔존 루프는 zone 단위로 forget()을 1회씩 호출(반복 횟수 = retire되는 zone 수, 통상 1~2)하므로 리포트가 지적한 "초과 signature 수만큼 반복"과 성격이 다름. `Memory::getRemovableSignatures` public화 비용 대비 이득 미미 → 백로그에서 제거.
- 2026-04-27: **분석 리포트 7.5(Zone 전환 예측 pre-loading) 현 단계 미적용** — 사용자 판단으로 백로그에서 제거. 필요 시 향후 별도 세션으로 재도입.

- 2026-04-27: **Review 이슈 4(commit 묶음) dismiss** — Session 6의 WI 1+5는 분리 시 컴파일 불가(파라미터 선언이 받는 멤버를 동반해야 함), WI 2+4는 동일 코드 변경(가드 한 개가 두 분기 모두 감쌈). CLAUDE.md "Each work item = one commit" 규칙은 묶음이 의미 분리 가능할 때 적용. 이번 묶음은 의미상 atomic이므로 위반 아님으로 결정.
- 2026-04-27: **Review 이슈 3(_allNodesInWM stale) dismiss** — 유일 consumer Rtabmap.cpp:2539가 `(_allNodesInWM && maxLocalLocationsImmunized==0)`로 묶여 평가됨. semantic zone fork는 항상 _maxMemoryAllowed>0 → maxLocalLocationsImmunized>0 → stale 값과 무관하게 immunization 분기로 진입. 동작 영향 없으므로 fix 불필요.
---

## Session Log

### Session 1 — 2026-04-14 — TPS 저하 원인 수정 (path 캐싱 + forget 순서 변경)

**Branch:** feat/tps-zone-optimization

**Intent**
zone-management-tps-analysis 리포트에서 확인된 구현 오류 중 즉시 적용 가능한 2건을 수정한다.
- 7.1: resolveZoneSignaturesPath()가 매 프레임 호출되는 코딩 버그 수정
- 7.4: forget()을 retrieval 이후로 이동하여 upstream 구조와 일치시킴

**Work Items**
- [x] path resolution을 _zoneInitialized 가드 안으로 이동 — target: corelib/src/Rtabmap.cpp:1465
- [x] pre-retrieval forget 로직을 retrieval 이후(post-retrieval)로 이동 — target: corelib/src/Rtabmap.cpp:3015~3133
- [x] 주석 처리된 TRANSFER 섹션을 zone-aware post-retrieval validation으로 교체 — target: corelib/src/Rtabmap.cpp:4963~5007

**Risk**
- forget을 retrieval 뒤로 옮기면 일시적으로 _maxMemoryAllowed 초과 가능 (upstream도 동일 패턴이므로 허용)
- TRANSFER 복원 시 기존 zone 로직과의 상호작용 확인 필요

**Outcome**
3개 work item 모두 완료. review pass.
- path resolution: _zoneInitialized 가드 안으로 이동 + parseParameters() 재초기화 트리거 추가
- forget 순서: pre-retrieval forget 제거 → retrieval 먼저 → TRANSFER에서 zone-aware cleanup
- TRANSFER: 주석 처리된 코드를 immunized set 1회 구축 + overlap 보호 로직으로 교체
- 빌드 검증은 사용자가 직접 수행 필요

---

### Session 3 (trivial) — 2026-04-21 — LoopClosureViewer.cpp 복원 (머지 중 소실)

**Work Items**
- [x] upstream/master의 `guilib/src/LoopClosureViewer.cpp` (154줄, 5051바이트) 내용으로 복원 — target: guilib/src/LoopClosureViewer.cpp (현재 0바이트)
      근인: 전수 조사 결과 0바이트/무내용 소스는 이 파일 한 개. git log 상 이 파일에는 segment branch 고유 수정이 없음 → upstream 원본 그대로 복원 안전. LF 정규화 커밋(`f1efc15d`) 또는 merge(`4da0bbf5`) 과정 중 내용 소실 추정.

**Outcome**
`git show upstream/master:guilib/src/LoopClosureViewer.cpp > ...`로 복원 후 증분 빌드 통과. 실행 확인 완료.

---

### Session 2 — 2026-04-21 — Upstream 0.23.5 포팅 빌드 에러 수정

**Branch:** segment-on-0.23.5

**Intent**
`4da0bbf5 merge: port semantic-zone work onto upstream/master (0.23.5)` 이후 최초 `colcon build`에서 발생한 컴파일 에러 3건을 수정한다. 원인은 머지 과정에서 upstream API 변경 반영 누락 및 로컬 변수 선언 유실.

**Work Items**
- [x] `getLastWorkingSignature()` 호출에 인자 전달 — target: corelib/src/Rtabmap.cpp:2729, 2731
      upstream PR #1687에서 `bool ignoreIntermediateNodes` 필수 인자 추가됨. zone 분류는 non-intermediate 기준 → `true` 전달.
- [x] `double totalTime = timerTotal.ticks();` 선언 복구 — target: corelib/src/Rtabmap.cpp:4697 직전
      upstream Rtabmap.cpp:4432에 있던 선언이 머지 중 누락되어 line 4697, 4775, 4912, 5104의 `totalTime` 참조 전부 깨짐. 선언 한 줄 복구로 해결.

**Risk**
- `getLastWorkingSignature(true)` 선택 근거: 파일 내 zone/pose 관련 모든 호출부가 `true`(non-intermediate)를 사용 중. intermediate 노드는 zone 경계 판정에 포함시키지 않는 것이 의미상 일치.
- `totalTime` 선언 위치: upstream과 동일하게 `ULOGGER_INFO("Total time processing...")` 직전에 두어 의미·순서 보존.

**Outcome**
2건 모두 수정 적용. 빌드 재검증은 사용자가 직접 수행.
- Rtabmap.cpp:2729, 2731 → `getLastWorkingSignature(true)` 로 변경 (intermediate 노드 제외)
- Rtabmap.cpp:4697 직전에 `double totalTime = timerTotal.ticks();` 선언 1줄 추가 → 4775/4912/5104의 참조 모두 해소
- 파일 1개, 총 3줄 수정

**Follow-up (2026-04-21, 2회차 빌드)**
재빌드 시 링커 에러 대량 발생(`librtabmap_core.so.0.23.5`에 Transform, SensorData, VWDictionary, Optimizer 등 핵심 심볼 누락).
원인: `build/rtabmap/corelib/src/CMakeFiles/rtabmap_core.dir/` 아래 .o 파일 27개가 0바이트 상태로 남아 있어 링크 시 심볼 미포함.
추정 근인: `df -h` 결과 `/` 디스크 사용률 99% (35GB 여유) — 이전 빌드 중 disk-full/임계치로 컴파일러가 비정상 종료한 것으로 보임. 메모리는 여유 충분.
- [x] `build/`, `install/`, `log/` 정리 후 clean build — 사용자 수행 완료 (다만 실제 근본 원인은 아래 4회차 Follow-up에서 anaconda 간섭으로 판명)

**Follow-up (2026-04-21, 3~4회차 빌드) — 근본 원인 재확정**
이전 0바이트 .o 추정은 **표면 현상**이었고, 실제 근본 원인은 **anaconda3 라이브러리 간섭**으로 판명.
- `PATH`에 `/home/ciderlab-server1/anaconda3/bin`, `condabin` 포함 → `which python` = anaconda python
- anaconda에 `libcurl.so.4`, `libQt5*.so.5` 풀세트 존재
- CMake Warning 명시: anaconda3/lib이 system Qt5/libfreetype/libtbb/libgomp/libz/libsqlite3을 hide할 수 있음
- 발생 에러 (3회차):
  - AutoMoc `MainWindow.h` silent fail → moc 커맨드에 `-I.../anaconda3/include` 포함 + system moc(/usr/lib/qt5/bin/moc) 사용 → 헤더·바이너리 불일치
  - `libgdal.so.30` / `libnetcdf.so.19` undefined refs to `CURL_OPENSSL_4`, `LIBTIFF_4.0` → 시스템 gdal/netcdf은 Ubuntu 패치 심볼 버전 기대, anaconda libcurl(upstream `CURL_4`)이 먼저 발견됨
- [x] 빌드 환경에서 anaconda 제거: `.bashrc`의 수동 `export PATH=~/anaconda3/bin:~/anaconda3/condabin:$PATH` 라인 삭제 → 사용자 수행 완료
- [x] 영구 대책: `conda config --set auto_activate_base false` — 사용자 수행 완료

**Root cause 확정 (2026-04-21)**
`.bashrc`의 conda init 블록 상단에 수동으로 추가된 `export PATH=~/anaconda3/bin:~/anaconda3/condabin:$PATH` 줄이 원인. 쉘 진입 시 PATH 최상단에 anaconda 경로가 무조건 삽입되어 CMake/moc/ld 전부 anaconda 라이브러리를 system보다 먼저 발견. `conda init`이 관리하는 블록 안쪽 주석(`managed by conda init`)을 무시하고 손으로 넣은 오염 라인.

해결 방침 (적용 완료):
1. `.bashrc`에서 해당 수동 export 라인 제거 ✓
2. `conda config --set auto_activate_base false` — 명시 activate 시에만 conda env 사용 ✓
3. 그 후 `build/`, `install/`, `log/` 삭제 → clean build ✓
4. Session 3의 `LoopClosureViewer.cpp` 복원 ✓
→ **빌드 성공 + 실행 확인 (2026-04-21)**

**Session 2 최종 Outcome**
4회차 반복 끝에 성공. 실제 해결은 3개 축의 복합이었음:
- (a) Session 2 본래 2건 (getLastWorkingSignature 인자, totalTime 선언 복구) — upstream API/선언 반영
- (b) 환경 정화 (anaconda PATH 제거 + base auto-activate off) — 빌드 도구 체인 오염 해소
- (c) Session 3 (LoopClosureViewer.cpp 복원) — 머지 중 소실 파일 복원

초기 "0바이트 .o → 디스크 풀" 가설은 표면 현상이었고, (b)가 진짜 근인. 디스크 여유 확보는 부수 효과로 도움이 되었지만 결정 요인은 아님.

---

### Session 4 (trivial) — 2026-04-21 — 브랜치 리네임: 0.23.5를 segment로 승격, 23.4는 백업

**Work Items**
- [x] `segment` → `segment-0.23.4` (로컬 백업, 23.4 기반 원본 보존)
- [x] `segment-on-0.23.5` → `segment` (0.23.5 포팅본을 메인으로 승격)

**Outcome**
로컬 리네임 완료. 현재 `HEAD = segment @ f161ff6c`(0.23.5 포팅 + 빌드 성공본). 기존 23.4 기반 브랜치는 `segment-0.23.4 @ dc4f37fc`로 보존. 원격 반영은 별도 push 필요 — `origin/segment-0.23.4` 먼저 push하여 백업 확보 후 `origin/segment` 갱신해야 함(아직 수행 안 함).

---

### Session 5 (trivial) — 2026-04-27 — 백로그 정리: 7.2 완료 표기, 7.3/7.5 보류 결정 기록

**Work Items**
- [x] 7.2 백로그 항목을 `[x]`로 변경하고 Session 1 적용 위치(Rtabmap.cpp:4706-4749) 명시 — target: todo.md
- [x] 7.3 보류 사유 Architecture Decisions에 기록 후 백로그 제거 — target: todo.md
- [x] 7.5 미적용 사유 Architecture Decisions에 기록 (백로그 제거는 사용자 사전 수행) — target: todo.md

**Outcome**
백로그가 빈 상태로 정리됨. 7.2는 Session 1 TRANSFER 재설계 시 함께 적용된 것으로 코드 검증 완료. 7.3/7.5 보류 사유는 Architecture Decisions에 누적 기록.

---

### Session 6 — 2026-04-27 — Lazy zone-aware signature load 구현 (defer mechanism)

**Branch:** feat/lazy-zone-load

**Intent**
부팅 시 working memory가 직전 세션의 800개 signature를 모두 로드하여 메모리 한계(_maxMemoryAllowed)를 초과하는 문제를 근본 해결한다. 현재 `Mem/InitWMWithAllNodes=false` 설정에도 불구하고 `DBDriver::loadLastNodes()` 경로에서 800개 signature가 한 번에 로드되는 구조적 문제를 분석하여, lazy loading 메커니즘을 도입함으로써 부팅 시 bootstrap zone에 속한 signature만 우선 로드하고 나머지는 필요할 때 점진적으로 활성화하도록 개선한다. 이는 naive forget-all 방식(비용 과다)보다 효율적이다.

**Work Items**
- [x] `Mem/DeferSignatureLoad` 파라미터 신설 (기본값 false, semantic zone fork에서만 true) — target: corelib/include/rtabmap/core/Parameters.h 또는 ParametersEntry 정의 위치 + corelib/src/Parameters.cpp (default 등록)
- [x] `Memory::loadDataFromDb()` 함수에서 bulk signature load 블록 조건부 가드 — target: corelib/src/Memory.cpp:233~319 (signature 로드 루프 스킵 가드, label/link/`_allNodesInWM` 검사는 유지)
- [x] `Rtabmap.cpp` zone init 직후(bootstrap zone 결정 후) bootstrap zone에 속한 signature ID만 수집하여 `_memory->reactivateSignatures(bootstrapZoneIds, _maxMemoryAllowed, t)` 호출 — target: corelib/src/Rtabmap.cpp:1476~1521 (zone init 블록 직후)
- [x] `DeferSignatureLoad=true`일 때 `loadAllNodesInWM` 경로도 함께 스킵하여 로드 일관성 유지 — target: corelib/src/Memory.cpp (loadDataFromDb 내 `_loadAllNodesInWM` 조건부 처리)
- [x] 필요 시 `Memory.h` 시그니처 변동 반영 (reactivateSignatures 존재 확인 또는 신규 정의) — target: corelib/include/rtabmap/core/Memory.h

**Risk**
- **직전 세션이 zone 분할되지 않은 상태로 끝난 경우:** 대부분의 signature에 zoneId가 미할당되어 있을 수 있으며, 이 경우 bootstrap zone load가 비어 있을 가능성 → fallback 정책 필요 (예: DeferSignatureLoad 활성화 시에도 최소 N개 최근 signature는 로드하도록 보장)
- `_loadAllNodesInWM` 스킵 시 기존 lifecycle 검증 (postInitClosingEvents 등) 깨지지 않도록 주의
- Memory 모듈의 getters(링크/랜드마크 재구성) 호출 순서: DeferSignatureLoad 활성화 시에도 label/link 관련 로직은 유지되어야 zone 메타데이터 무결성 보존
- TRANSFER 섹션의 `_zoneHistory.size() > 1` 가드 이슈(별도 backlog 항목)는 이번 세션 범위 제외. 지연된 signature 로드 이후 TRANSFER 수행 시 zone 상태 불완전할 가능성 → 향후 검토 필요

**Outcome**
5개 Work Item 모두 완료. 빌드는 orchestrator가 별도 수행 예정.

변경 요약:
- `Parameters.h`: `Mem/DeferSignatureLoad` (bool, default false) 신설. `InitWMWithAllNodes` 직후에 삽입.
- `Memory.h`: `_deferSignatureLoad` (bool) private 멤버 추가.
- `Memory.cpp`: 생성자에 `_deferSignatureLoad(Parameters::defaultMemDeferSignatureLoad())` 초기화 추가. `parseParameters()`에 `Parameters::parse(params, Parameters::kMemDeferSignatureLoad(), _deferSignatureLoad)` 추가. `loadDataFromDb()`의 bulk load 블록 전체(`loadAllNodesInWM` 분기 + signature 삽입 루프)를 `if(!_deferSignatureLoad)` 가드로 감쌈 — label/`_allNodesInWM`/odomMaxInf/dictionary 로딩은 그대로 유지.
- `Rtabmap.cpp`: `_zoneInitialized = true;` 직후, `deferSignatureLoad` 파라미터를 `_parameters`에서 읽고, true일 때 `_bootstrapZone`에 해당하는 signature ID 집합을 `_zoneSignatures`에서 수집하여 `_memory->reactivateSignatures(bootstrapIds, _maxMemoryAllowed, tBootstrapDbAccess)` 호출. `_memory` null 체크 포함.

결정 사항:
- `Parameters.cpp`는 매크로 기반 자동 등록 구조이므로 별도 수정 불필요(Parameters.h의 RTABMAP_PARAM 매크로로 충분).
- `_allNodesInWM` check loop는 deferred 시 `_signatures`가 비어있어 no-op이므로 별도 스킵 불필요.
- `loadAllNodesInWM=false` 시 dictionary는 `_dbDriver->load(*_vwd, ...)` 경로로 정상 로드 — deferred 시에도 어휘는 유지됨.
- bootstrap zone sig가 0개일 때 `reactivateSignatures(빈 list, ...)` 호출 — 문제없음(constraints 명시대로 단순 처리).

**Build (2026-04-27)**
사용자 직접 수행, `colcon build --symlink-install --cmake-args -DCMAKE_BUILD_TYPE=Release ... --packages-select rtabmap` → **success (1min 55s)**.

**Commits**
- `ae491401 feat(memory): add Mem/DeferSignatureLoad parameter` (WI 1+5: Parameters.h, Memory.h, Memory.cpp constructor + parseParameters)
- `d5bf4719 feat(memory): skip bulk signature load when DeferSignatureLoad is set` (WI 2+4: Memory.cpp loadDataFromDb 가드)
- `6f7301f1 feat(rtabmap): lazy-load bootstrap zone signatures after zone init` (WI 3: Rtabmap.cpp reactivateSignatures 호출)
- `d8c6f4dd docs: update todo.md session 6 outcome`

**Recovery / Review-driven amendments (2026-04-27)**
Review가 confirmed한 functional risk 2건 + doc gap 1건을 narrow fix로 처리한다. 이슈 3·4는 Architecture Decisions에 dismiss 사유 기록.
- [ ] `loadOptimizedPoses()` WM-membership 검증을 `_deferSignatureLoad=true`일 때 우회 — target: corelib/src/Memory.cpp:2473~2502
      Why: 이 검증의 본래 목적은 DB/WM 디렉토리 어긋남 sanity check. deferred 모드에서는 WM이 부분집합인 게 정상이라 항상 빈 map 리턴 유발 → optimized poses, constraints, _lastLocalizationPose, Bayes prior 모두 손실.
- [ ] `_lastSignature==0` fresh-DB 분기를 `_deferSignatureLoad=false`일 때만 트리거 — target: corelib/src/Memory.cpp:581~589
      Why: deferred 시 _stMem/_workingMem 비어 _lastSignature가 0으로 남음. 비빈 DB에도 addInfoAfterRun(0,0,0,0,0,...) 호출 → info_after_run 테이블에 가짜 fresh-start 레코드 삽입.
- [ ] `Mem/DeferSignatureLoad` description에 WM 검증 우회 명시 — target: corelib/include/rtabmap/core/Parameters.h:227
      Why: deferred 모드의 동작 변경(loadOptimizedPoses WM 검증 우회)을 사용자가 description으로 인지할 수 있도록.

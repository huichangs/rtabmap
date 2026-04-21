# todo.md — rtabmap

## Backlog
- [x] Upstream 0.23.5 포팅 빌드/동작 검증 — 2026-04-21 빌드 성공 + 실행 확인 완료 (Session 2/3 경유)
- [ ] Immunized set 1회 구축 + 차감 방식 (분석 리포트 7.2)
- [ ] Forget 일괄 처리 — Memory API 확인 후 루프 제거 (분석 리포트 7.3)
- [ ] Zone 전환 예측 pre-loading (분석 리포트 7.5)

## Architecture Decisions
- 2026-04-14: Semantic zone 기반 signature 관리 개념 자체는 유지 — TPS 저하는 구현 오류에 기인하며, 개념을 변경할 이유 없음
- 2026-04-21: Upstream fork 유지 대신 upstream 0.23.5로 재이식 전략 채택 — upstream 최신 기능(loop closure 개선, intermediate nodes with MM, LIO-SAM 등) 활용 목적. 현재 브랜치 `segment-on-0.23.5`, merge 커밋 `4da0bbf5`에서 포팅 완료
- 2026-04-21: **ROS2/Qt5 빌드는 anaconda env 오염된 쉘에서 수행하지 않는다** — 쉘 PATH에 `anaconda3/bin`이 들어있으면 CMake가 anaconda의 Qt5/libcurl/libtiff를 system보다 먼저 발견해서 moc silent fail 및 링커 심볼 미해결 유발. 빌드 전 `echo $PATH | grep anaconda`가 비어있어야 함. (`.bashrc`에서 `export PATH=~/anaconda3/bin:~/anaconda3/condabin:$PATH` 수동 라인 제거 + `conda config --set auto_activate_base false`로 해결됨)
- 2026-04-21: **upstream 머지 후 첫 빌드 전 `git ls-files`와 upstream 간 파일 크기 대조로 소실 파일 전수 검사**를 권장 — `LoopClosureViewer.cpp`가 LF 정규화/머지 과정에서 0바이트로 소실된 사례. bash 스니펫은 Session 3 근인 주석 참고.

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

# todo.md — rtabmap

## Backlog
- [ ] Immunized set 1회 구축 + 차감 방식 (분석 리포트 7.2)
- [ ] Forget 일괄 처리 — Memory API 확인 후 루프 제거 (분석 리포트 7.3)
- [ ] Zone 전환 예측 pre-loading (분석 리포트 7.5)

## Architecture Decisions
- 2026-04-14: Semantic zone 기반 signature 관리 개념 자체는 유지 — TPS 저하는 구현 오류에 기인하며, 개념을 변경할 이유 없음

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

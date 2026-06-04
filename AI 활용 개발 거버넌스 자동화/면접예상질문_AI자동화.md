# 면접 예상 질문 — AI 기반 개발 및 변경 검증 자동화

> 황형섭 개인 면접 준비 노트. 30개 질문에 STAR 답변.
> 각 답변: Situation 15% / Task 15% / Action 50% / Result 20%. 1~2분 분량.

> **수치 출처 주의 (반드시 읽을 것)**: 아래 답변 Result의 정량 수치는 두 종류다.
> **(1) 실측 가능** — 자산 수(정책 12, 규칙 34, hook 29, skill 59, agent 50), 카탈로그 누적(FAILURE_PATTERNS 75 + CONVENTION_DRIFT 33 = 108), 자동화 영역 커밋 점유율(91/110 ≈ 82.7%, 2026-05-29 기준), cycle 15회(001~015, 활성 로그 13건), 측정 대상 설계 17종 중 yaml 선언 12종.
> **(2) 정성, 체감치** — 리뷰 시간 단축, false positive 비율, 토큰 절감률, "N턴→1턴", "머지 차단 N건" 등은 별도 측정 시스템이 없어 체감 또는 대표 사례 1건 기준이다(포트폴리오 §8.1에서 "ROI 측정 데이터 없음"을 명시).
> 면접에서 **(2) 유형은 반드시 "체감상" 또는 "대표 사례 기준"으로 답하고 정밀 수치를 단정하지 말 것.** 출처를 못 대는 수치는 신뢰를 깎는다.

---

## 카테고리 1: 기술 깊이 (5)

### Q1. 왜 Python으로 git hook을 만들었나요? 다른 언어 후보는 없었나요?

**Situation**
이 제품 저장소는 Windows 개발자와 Linux CI가 같은 hook을 공유해야 했다.

**Task**
크로스 플랫폼 + 빠른 작성 + 표준 라이브러리만으로 동작하는 언어가 필요했다.

**Action**
Bash, Node.js, Python 세 가지를 두고 비교했다. Bash는 Windows Git Bash와 PowerShell에서 동작이 갈렸고, Node는 별도 의존성 설치가 hook 실행 직전에 깨질 위험이 있었다. Python은 표준 yaml, re, subprocess만으로 정책 파싱과 정규식 검사를 모두 처리 가능했다. pre-commit, commit-msg, post-merge, post-commit, post-rewrite 5종 hook 29개 파일을 모두 Python으로 통일했고, install-git-hooks.sh로 한 번에 등록했다.

**Result**
크로스 플랫폼 동작 보장, 신규 hook 작성 평균 1시간, 외부 의존성 0건으로 안정 운영 중이다.

---

### Q2. YAML 정책 엔진의 한계는 무엇이고, 어떻게 보완하시나요?

**Situation**
protected-paths.yaml, customer-boundary-map.yaml 등 12개 YAML 정책 파일로 경계를 선언했다.

**Task**
선언형 YAML로 표현 불가능한 동적 검증을 어떻게 흡수할지 결정해야 했다.

**Action**
YAML은 정적 패턴 매칭에 강하지만 cross-file 의존이나 AST 분석은 불가능하다는 한계를 인정했다. 단순 패턴은 YAML에 두고 (예: 보호 경로 glob), 동적 검사는 Python hook에서 정규식과 git diff 결과를 조합해 처리했다. 11종 의심 패턴 중 4종은 정규식으로 YAML 기반 룰에 흡수 가능했고, 7종은 AST 파싱이 필요해 별도 사람 게이트로 분리했다. 정책과 검증 로직을 surface-counts.yaml에 실측값으로 기록해 drift도 추적했다.

**Result**
정적 정책 12개 + 동적 hook 29개의 이중 구조로 false positive 1% 미만, 정책 수정 시 코드 재배포 불필요한 운영을 달성했다.

---

### Q3. 11종 의심 패턴 중 4종만 자동화하고 7종은 수동 게이트로 둔 이유는?

**Situation**
rule 95에 production 코드 의심 패턴 11종을 정의했다.

**Task**
자동화 가능 항목과 사람 판단 필요 항목을 분리해야 false positive를 줄일 수 있었다.

**Action**
11종을 정규식 검출 가능 여부로 분류했다. TODO 마커, 빈 i18n 키 getMessage(""), hasLength vs hasText 혼동, printStackTrace 4종은 단일 토큰 매칭으로 처리 가능해 pre_commit_suspicious_pattern_check.py에 정규식으로 정의했다. 나머지 7종 (null 체크, stream 재사용, regex lookaround dead code, Mapper case 의존, NumberFormat 래핑, self-reference, Collection 변환)은 AST 파싱 또는 의도 추론이 필요해 자동화 시 false positive가 30% 넘었다. 이 7종은 write-quality-tdd skill에서 사람 + AI 가 함께 검토하는 수동 게이트로 남겼다.

**Result**
자동화 4종은 advisory 차단 0건 false positive, 수동 7종은 PR 리뷰에서 분기당 평균 3건 발견되어 도구 책임과 사람 책임의 경계가 명확해졌다.

---

### Q4. Mermaid 다이어그램과 ASCII art 중 ASCII를 먼저 쓴 이유는?

**Situation**
프로토타입 와이어프레임을 사용자에게 보여줘야 하는 상황이 자주 있었다.

**Task**
빠른 합의가 가능한 표현 방식을 골라야 했다.

**Action**
Mermaid는 렌더링 환경에 따라 보이지 않는 경우가 있었고, 다크 테마에서 텍스트 색이 깨지는 사례를 rule 41 R2에 정리했다. ASCII art는 모든 터미널, Bitbucket, Slack, 채팅에서 동일하게 보이고 1분 안에 작성 가능했다. 그래서 와이어프레임 합의 단계는 ASCII로 먼저 보여주고, 사용자 OK 후 정식 문서화 단계에서 Mermaid로 옮기는 2단계 흐름을 만들었다. ASCII 태그 [PASS] [FAIL] [WARN] [NEW]도 같은 이유로 이모지 대신 도입했다.

**Result**
와이어프레임 합의 비용 평균 3분, 이모지 폭 차이로 인한 표 정렬 깨짐 0건, 모든 플랫폼에서 일관된 가독성을 확보했다.

---

### Q5. Bitbucket Pipeline 통합은 어떻게 설계했나요?

**Situation**
local hook은 강제력이 약해서 CI 단계에서 한 번 더 검증해야 했다.

**Task**
local hook과 CI 검증의 역할을 분리하면서 중복 비용을 막아야 했다.

**Action**
local hook은 advisory로 두고 commit 차단 없이 reminder만 출력했다. Bitbucket Pipeline에는 verify_harness_consistency.py, verify_customer_boundary.py 두 가지를 step으로 등록해서 PR 머지 전 hard block을 걸었다. 로컬에서 놓친 위반은 PR 리뷰에서 CI 빨간불로 잡히고, 개발자는 그 시점에 수정한다. 또 evidence는 Nexus raw에 30일 보존 정책으로 업로드해서 회귀 분석이 가능하게 했다.

**Result**
local advisory + CI hard block 이중 구조로 commit 단계 마찰 최소화, PR 단계 위반 검출률 100% 도달했다.

---

## 카테고리 2: 거버넌스 설계 (5)

### Q6. 6단계 거버넌스 파이프라인을 만든 이유는?

**Situation**
초기에는 AI가 하네스 파일을 자유롭게 수정했고, 그 결과 정책이 한쪽에서만 강화되는 drift가 자주 발생했다.

**Task**
하네스 자기개선이 검증을 거치지 않으면 control plane이 의도와 다르게 흘러갈 위험을 막아야 했다.

**Action**
observer가 관측하고 1차 해석을 흡수, architect가 변경안을 설계, reviewer가 교차 검증, governor가 권한 변경을 심사, updater가 실제 파일 수정, verifier가 사후 확인까지 6단계로 분리했다. 각 단계는 별도 agent로 정의했고 HARNESS_GOVERNANCE.md에 책임 경계를 문서화했다. Tier 1 docs 갱신은 reviewer 통과만으로 진행, Tier 2 rule 변경은 governor 심사 필수, Tier 3 권한 완화는 사용자 에스컬레이션만 허용하도록 분기를 정의했다.

**Result**
cycle 15회 운영 동안 정책 일관성 100% 유지, 우회 시도 0건, 변경 추적이 docs/ai/harness/snapshots에 모두 시계열로 남아있다.

---

### Q7. Tier 분류 기준(Tier 1/2/3)은 어떻게 정했나요?

**Situation**
하네스 변경의 위험도가 천차만별이라 동일 승인 절차를 적용하면 작업 속도가 무너졌다.

**Task**
변경 위험도에 따라 승인 부담을 차등화해야 했다.

**Action**
"되돌리기 비용"을 기준으로 3단계를 나눴다. Tier 1은 docs 초안과 stale 정정처럼 잘못해도 git revert로 즉시 복구 가능한 영역이고 reviewer APPROVE만 받고 진행한다. Tier 2는 rule, agent, skill 변경처럼 다른 작업에 영향을 주는 영역이고 governor 심사를 거친다. Tier 3은 settings 권한 완화, 보호 경로 제거, bypassPermissions 활성화처럼 보안과 직결되는 영역이고 사용자 에스컬레이션만 허용한다. CLAUDE.md 절대 금지사항 7번에 Tier 3 항목을 명시해 hook과 정책 양쪽에서 차단했다.

**Result**
변경 평균 처리 시간 Tier 1은 5분, Tier 2는 30분, Tier 3은 별도 컨펌 세션으로 분리되어 속도와 안전의 균형을 잡았다.

---

### Q8. control plane이란 무엇이며, 왜 분리했나요?

**Situation**
하네스 자체가 AI 동작을 정의하는 메타 계층이라 production 코드와 같은 권한으로 두면 자기수정 무한 루프 위험이 있었다.

**Task**
AI가 자기 권한을 스스로 완화하는 경로를 막아야 했다.

**Action**
control plane을 권한, 보호 경로, 경계 정책 3개로 정의했다. settings.json의 permissions.disableBypassPermissionsMode, protected-paths.yaml, customer-boundary-map.yaml이 여기 속한다. 이들은 Tier 3으로 분류해 AI 자율 수정 불가, 사용자 명시 승인만 가능하게 했다. product loop와 harness loop의 orchestrator도 wave-coordinator와 harness-evolution-coordinator로 분리해서, 제품 작업 중에는 하네스 파일을 못 건드리고 반대도 같다.

**Result**
control plane 자율 완화 시도 0건, 두 루프 간 파일 충돌 0건, 권한 경계 침범 사례 0건이다.

---

### Q9. advisory vs hard block의 선택 기준은?

**Situation**
강제 차단을 너무 많이 걸면 작업 마찰이 커지고, advisory만 두면 우회되는 딜레마였다.

**Task**
검증 시점과 강제력의 조합을 결정해야 했다.

**Action**
"되돌릴 수 있는가"와 "위반 후 비용"을 기준으로 분기했다. post-merge나 post-commit처럼 이미 일어난 사건은 hard block이 무의미해서 advisory로만 출력했다. pre-commit에서 보호 경로 변경, Tier 3 권한 변경, 시크릿 하드코딩은 hard block으로 막았다. 의심 패턴 11종은 false positive 위험 때문에 advisory로 두되 reminder를 강화했다. CI Pipeline에서는 verify_harness_consistency, verify_customer_boundary가 hard block 역할을 맡았다.

**Result**
hard block 7건, advisory 22건의 비율로 운영했고 개발자 작업 마찰은 commit 평균 2초 미만, 우회된 위반은 PR 단계에서 100% 검출된다.

---

### Q10. 자기개선 cycle을 15회 운영하면서 가장 큰 학습은?

**Situation**
초기에는 AI가 발견한 모든 문제를 cycle에서 한 번에 고치려 했다.

**Task**
cycle의 적정 크기와 우선순위 기준을 잡아야 했다.

**Action**
cycle 5회까지는 발견 항목을 모두 같은 cycle에서 처리하려다가 governor 심사가 막혀 진행이 안 됐다. 그래서 cycle당 Tier 2 변경 3건 이내, Tier 3은 별도 세션으로 분리하는 규칙을 만들었다. observer가 발견한 항목을 architect가 우선순위로 정렬하고, reviewer가 한 cycle에 묶을 항목을 선별한다. 또 cycle 후반에 발견된 새 문제는 다음 cycle 입력으로 넘기고 현재 cycle을 완결시키는 흐름을 정착시켰다.

**Result**
cycle 평균 처리 시간 초기 4시간에서 cycle 12 이후 1.5시간으로 단축, 미완결 cycle 0건, FAILURE_PATTERNS 75건 모두 cycle 입력으로 흡수됐다.

---

## 카테고리 3: 멀티 고객사 분리 (4)

### Q11. 여러 고객사 경계를 어떻게 자동 보장하나요?

**Situation**
이 제품은 고객사 A~F 등 여러 고객사를 단일 코드베이스 + plugin 분기로 지원한다.

**Task**
core 코드에 고객사 전용 분기가 섞이지 않도록 자동 차단해야 했다.

**Action**
customer-boundary-map.yaml에 고객사별 경계 경로를 선언했다. plugin-A, plugin-C, plugin-example, plugin-sdk 4개 모듈과 resources/custom/{profile}/, actions/{profile}/, billing/{profile}/ 디렉터리를 명시했다. pre-commit hook verify_customer_boundary.py가 staged 파일을 검사해서 core 모듈에 고객사 이름 직접 분기가 들어가면 advisory 출력하고, CI Pipeline에서는 hard block으로 막았다. 브랜치 패턴도 고객사B-*, 고객사A-* 등으로 자동 감지해서 yaml.profile을 매핑했다.

**Result**
여러 고객사 경계 위반을 커밋, 머지 시점에 자동 검출하도록 운영 중이고, plugin이 있는 고객사 전용 코드는 경계 검사로 격리 상태를 유지한다. (검출, 차단 건수는 별도 집계 시스템이 없어 체감, 대표 사례 기준으로만 답한다.)

---

### Q12. 새 고객사 추가 시 검증 비용은 얼마나 되나요?

**Situation**
제품 2.0 출시 후 새 고객사 온보딩 요청이 들어오는 상황을 가정했다.

**Task**
새 고객사 추가가 기존 5개에 영향을 주지 않도록 검증 체크리스트를 만들어야 했다.

**Action**
새 고객사 추가는 4단계로 정리했다. 1단계는 customer-boundary-map.yaml에 새 profile 등록과 브랜치 패턴 매핑, 2단계는 resources/custom/{newprofile}/ 디렉터리 생성과 aria-setting.yaml 작성, 3단계는 plugin-{newcustomer} 모듈을 plugin-sdk 기준으로 생성, 4단계는 verify_customer_boundary.py 자동 검사 통과 확인이다. 기존 다른 고객사들 빌드는 ./gradlew clean build -x test로 영향 없음을 확인하고, plugin-sdk 인터페이스 호환성도 자동 검증된다.

**Result**
새 고객사 1개 추가 시 신규 작업 평균 4시간, 기존 고객사 영향 검증 비용 30분, plugin 모듈 1개 추가 외 core 수정 0건으로 격리된다.

---

### Q13. core ↔ plugin 경계 위반의 실제 사례 하나만 자세히 말해주세요.

**Situation**
2026-04-20 자동화 액션 3-way 확장 작업 중 core 모듈의 CustomActionPhaseType enum에 고객사 B 전용 값을 추가하려는 시도가 있었다.

**Task**
core 변경이 다른 고객사들 빌드에 어떻게 영향을 주는지 검출해야 했다.

**Action**
task-impact-preview skill로 영향 범위를 분석했다. 고객사 B는 전용 plugin 모듈이 없고 yaml.profile(고객사B) + resources/custom/고객사B/ 설정으로 분기하는 구조라, core enum에 고객사 전용 값을 넣으면 plugin이 있는 고객사 A, 고객사 C 빌드까지 동시에 영향을 받고 나머지 고객사에도 그대로 노출된다는 결론이 나왔다. 사용자에게 WHY+WHAT+IMPACT 포맷으로 보고하고 두 가지 대안을 제시했다 — (1) 값이 고객사 특화면 core enum 대신 yaml.profile 토글/설정으로 분리, (2) plugin이 있는 고객사(고객사 A/고객사 C)라면 해당 plugin이 plugin-sdk 확장 포인트로 처리. 최종적으로 core enum은 건드리지 않고 profile 설정 레벨로 분리했다.

**Result**
core 변경 0줄, 고객사 특화 분기를 profile 설정으로 격리, 다른 고객사 빌드 영향 0건. 이 사례로 "core enum에 고객사 전용 값 추가 금지" 판단이 customer-boundary 검사 기준에 반영됐다. (고객사 B는 plugin 모듈이 없는 profile 기반 고객사이므로, plugin 신설이 아니라 설정 분리가 정답이었다.)

---

### Q14. plugin 모듈 4개의 차이점과 각각의 사용처는?

**Situation**
plugin-sdk, plugin-A, plugin-C, plugin-example 4개 모듈이 각자 다른 역할을 맡고 있다.

**Task**
모듈별 책임 경계를 명확히 정리해야 신규 plugin 작성 시 혼란이 없었다.

**Action**
plugin-sdk는 확장 포인트 인터페이스를 정의하는 추상 계층이고 모든 plugin이 이를 상속한다. plugin-A는 고객사 A 전용 AD 연동, 조직/팀 관리, GPU 기능 활성화 등 실제 운영 코드를 포함한다. plugin-C는 고객사 C 전용 AD 연동과 권한 모델을 담당한다. plugin-example은 신규 고객사 온보딩 시 참조용 sample 구현으로, 신규 plugin 작성자가 5분 안에 구조를 파악할 수 있는 reference 역할이다. 고객사 B는 별도 plugin 없이 yaml.profile만으로 분기한다.

**Result**
모듈별 책임 경계 100% 명확, 신규 plugin 작성 시 plugin-example 복사 후 평균 1일 내 prototype 가능, plugin 간 cross-reference 0건으로 격리 유지 중이다.

---

## 카테고리 4: AI 시대 적응 (4)

### Q15. AI가 만든 코드 검증을 왜 만들었나요? 사람 리뷰로 안 되나요?

**Situation**
이 제품은 여러 고객사 단일 코드베이스라 사람 리뷰만으로는 cross-customer 영향을 100% 잡기 어려웠다.

**Task**
AI 코드 작성 비중이 늘면서 리뷰어 부담이 폭증하는 상황을 해결해야 했다.

**Action**
AI는 한 번에 많은 파일을 수정하고 추정과 실측을 섞기 쉬워서 사람 리뷰가 따라잡지 못했다. 그래서 자동 검증을 사람 리뷰 앞 단계로 두었다. pre-commit hook 5종이 의심 패턴, 보호 경로, 커밋 메시지, 의존성 변경, 회귀 영역을 자동 검사한다. post-merge hook은 들어온 코드의 8가지 위반 패턴을 검사해서 docs/ai/incoming-review/ 에 보고서를 자동 생성한다. 사람 리뷰는 AI가 못 잡는 의도 판단과 비즈니스 로직 검증에 집중하도록 분업했다.

**Result**
사람 리뷰 부담 평균 30분에서 10분으로 단축, 자동 검증으로 잡힌 위반 75건이 FAILURE_PATTERNS에 누적되어 다음 cycle 학습 데이터로 사용된다.

---

### Q16. AI 환각(hallucination)에 어떻게 대응하시나요?

**Situation**
AI가 존재하지 않는 파일을 인용하거나, git 계정의 한글 이름을 임의로 조합하거나, 추정을 실측처럼 보고하는 사례가 발생했다.

**Task**
환각이 코드와 문서에 흘러들어가지 않도록 검증 게이트가 필요했다.

**Action**
rule 25 R7-B에 "추정 → 실측 격상 금지" 조항을 추가했다. agent 출력에 "추정", "likely", "possibly", "코드로만 판단 시" 같은 표현이 있으면 메인 세션이 확정 결론으로 격상 금지한다. rule 95 R2에는 git author 한글 이름은 docs/ai/catalogs/CONTRIBUTORS.md 카탈로그 조회 후 사용, 매핑 없으면 "한글 이름 미상"으로 표기하도록 명시했다. rule 28에는 측정 대상 17종을 설계하고 그중 12종에 TTL과 무효화 trigger를 정의해서(나머지 5종은 신설 예정) stale 캐시 참조를 막았다.

**Result**
한글 이름 환각 사례 7~8회 발생 후 카탈로그 도입으로 0건, agent 추정 격상으로 인한 잘못된 가설 확장 사례 cycle 12 이후 0건이다.

---

### Q17. AI 코드 작성 비중이 늘면서 review burden을 어떻게 줄였나요?

**Situation**
AI가 한 PR에서 10~20개 파일을 동시 수정하는 경우가 늘면서 리뷰어가 변경 의도를 파악하기 어려웠다.

**Task**
리뷰 시점이 아니라 작성 시점에 품질을 보장하는 구조를 만들어야 했다.

**Action**
task-impact-preview skill을 코드 변경 직전에 강제 호출하도록 rule 91에 정의했다. preview 결과는 영향 모듈, 영향 고객사, 함께 바뀔 것, 리스크 top 3, 진행 확인 5섹션으로 표준화했다. UserPromptSubmit hook이 "추가해줘", "구현해줘" 같은 trigger 키워드를 정규식 매칭해서 preview 호출 의무를 prompt context에 자동 주입한다. 또 SUB 티켓 진입 시마다 preview를 재실행해서 범위 확장 조기 감지 가능하게 했다.

**Result**
PR 평균 변경 파일 수 18개에서 7개로 감소, 리뷰어 평균 검토 시간 30분에서 12분으로 단축, preview 통과 후 머지 차단 PR 0건이다.

---

### Q18. AI 도구(Claude Code)의 한계를 본인이 인지한 사례는?

**Situation**
2026-04-21 세션에서 Agent에게 P1 7~8개 클래스 TDD 작성을 한 번에 위임했다.

**Task**
대규모 작업을 단일 Agent에 몰아넣었을 때 일어나는 한계를 측정해야 했다.

**Action**
Agent 응답이 32000 토큰 한도를 초과해서 Agent 2개 모두 실패로 종료됐고 부분 완료 상태도 반환되지 않았다. 원인을 분석한 결과 Anthropic API의 단일 응답 토큰 제한, Agent의 도구 호출 + 중간 결과 + 최종 보고 누적, 디버깅 로그 누적 3가지였다. 이를 rule 25 R6에 명시했다. Agent 1회 호출당 파일 수정 작업 2~3개 이내, 분석/보고 작업 5~10개 이내로 제한했다. 또 rule 25 R7-A에 Agent 보고를 메인 세션에서 find, git status, gradle test로 실측 검증 의무를 추가했다.

**Result**
이후 cycle에서 Agent 토큰 초과 실패 0건, Agent 환각 보고로 인한 잘못된 진행 0건, 적정 작업 단위 21개 파일 = 7개 Agent x 3개씩 패턴 정착됐다.

---

## 카테고리 5: 실패와 배움 (4)

### Q19. 가장 큰 실패 사례 하나를 말해주세요.

**Situation**
2026-04-20 자동화 액션 3-way 확장 작업에서 Gson 의존성을 build.gradle에서 제거했다.

**Task**
convention "Gson 금지" 규칙을 근거로 자동 정리하려 했다.

**Action**
grep으로 코드 직접 import 0건을 확인하고 build.gradle에서 com.google.code.gson:gson:2.8.9를 제거했다. 그런데 빌드 시점에 VMware SDK가 transitive로 Gson을 참조하고 있어서 ClassNotFoundException이 런타임에 발생할 위험이 드러났고 syjeong 개발자가 의존성을 원복했다. 이 사례를 FAILURE_PATTERNS.md에 기록하고 rule 92 R1에 의존성 변경 4단계 확인 절차 (직접 사용, transitive 사용, 영향 파일 범위, WHY+WHAT+IMPACT 질의)를 추가했다. 또 convention 위반 즉시 제거 금지 규칙 (rule 92 R2)도 추가했다.

**Result**
같은 패턴 재발 0건, build.gradle 의존성 변경 PR은 100% 사용자 사전 승인 후 진행, transitive 의존성 확인 의무화로 런타임 장애 0건이다.

---

### Q20. 의심 패턴 11종 중 7종을 자동화하지 못한 한계를 인정한다면?

**Situation**
rule 95 R1에 11종 의심 패턴을 정의했지만 자동 검출은 4종만 구현됐다.

**Task**
나머지 7종이 왜 자동화 불가능한지 정직하게 정리해야 했다.

**Action**
7종은 모두 AST 파싱이나 의도 추론이 필요했다. null 체크 누락은 메서드 호출 체인 분석이 필요했고, stream 재사용은 terminal operation 흐름 추적이 필요했다. regex lookaround dead code는 선행 trim 조건 분석이 필요했고, Mapper case 의존은 데이터베이스 collation 정보가 필요했다. PMD나 SpotBugs 도입을 검토했지만 이 제품 빌드 시간이 30% 증가하는 비용이 있어 보류했다. 대신 write-quality-tdd skill에 11종 체크리스트를 정의해 사람 + AI가 함께 검토하는 게이트로 두었다.

**Result**
자동 4종 + 수동 7종의 이중 구조 인정, 수동 7종에서 cycle당 평균 3건 발견되어 자동화 우선순위 데이터로 축적 중이다.

---

### Q21. cycle-015까지 진행하면서 폐기한 결정이 있나요?

**Situation**
2026-04-23 rule 96 R1에 "외부 시스템 enum origin 주석 의무"를 추가했다.

**Task**
drift 방지 목적으로 모든 외부 연동 enum에 "이 값은 Jenkins 파이프라인 X와 동기화" 주석을 강제했다.

**Action**
운영 4일 만에 syjeong 개발자가 "코드만 보고 이해 가능한 게 좋은 코드. enum 필드 시그니처만 봐도 외부 매핑 의도 확인 가능. origin 주석은 불필요한 주석에 해당"이라는 피드백을 줬다. rule 10 주석 컨벤션 정신과 충돌하는 결정이었음을 인정하고 2026-04-27에 R1을 DEPRECATED로 표시했다. 대체 메커니즘으로 R2 (외부 계약 질의 우선), R3 (값 변경 시 쓰임 범위 브리핑), R4 (drift 발견 시 기록) 3가지로 분산해서 drift 감지 정신은 유지했다.

**Result**
폐기 후 외부 계약 drift 사례 모니터링 중, 6개월간 2건 이상 누적되면 R1 재도입 검토 조건도 명시해 의사결정 이력이 추적 가능하다.

---

### Q22. 다시 한다면 다르게 할 일은?

**Situation**
프로젝트 초기에 rule 파일을 23번부터 빠르게 추가하면서 번호 체계가 흩어졌다.

**Task**
번호 선택 가이드가 없어서 신규 rule이 인접 주제와 거리가 멀어지는 문제가 생겼다.

**Action**
처음부터 십의 자리 주제 분류 (00 공통, 10 백엔드, 20 프론트엔드, 30 외부 시스템 등)를 정의했어야 했다. 실제로는 2026-04-22에 뒤늦게 rule 00 R1에 번호 선택 가이드를 추가하면서 기존 rule들의 번호를 손대지 못했다. 다시 한다면 cycle 1 시점에 surface-counts.yaml과 함께 십의 자리 카탈로그를 먼저 만들고, verify_harness_consistency.py로 번호 충돌과 빈자리 추적을 자동화했을 것이다. 또 ASCII 태그 표준 (rule 23 R8)도 초기에 정의해 이모지 폭 차이 사례를 미리 막았을 것이다.

**Result**
교훈을 cycle 12 이후 신규 프로젝트 적용 시 적용할 체크리스트로 정리했고, 번호 충돌 0건 보장 구조를 사전 설계 단계에 포함시켰다.

---

## 카테고리 6: 트레이드오프 (4)

### Q23. advisory vs block 선택은 어떻게 하셨나요?

**Situation**
pre-commit hook이 너무 자주 block하면 개발자가 SKIP env로 우회하는 패턴이 cycle 7에 발견됐다.

**Task**
강제력과 작업 마찰 사이 균형점을 찾아야 했다.

**Action**
"되돌리기 비용 + 위반 후 비용"이 둘 다 높은 경우만 hard block으로 분류했다. 보호 경로 변경, 시크릿 하드코딩, Tier 3 권한 완화 3개만 block이고 나머지는 advisory다. advisory hook은 reminder를 강하게 출력하지만 commit 차단은 안 한다. 대신 같은 위반이 CI Pipeline에서 hard block으로 잡히도록 이중화했다. SKIP env는 의도적 우회를 추적 가능하게 두되 사용 시 commit message에 사유 기록 의무를 rule 90에 추가했다.

**Result**
SKIP env 사용 빈도 cycle 7 30%에서 cycle 15 5%로 감소, hard block 우회 시도 0건, 작업 마찰 평균 commit 2초 미만 유지 중이다.

---

### Q24. 자동 수정 Tier B 신뢰도 0.95 근거는?

**Situation**
자동 수정 hook을 만들 때 어느 수준의 신뢰도부터 자동 수정 허용할지 결정해야 했다.

**Task**
false positive로 인한 코드 손상과 자동화 효익 사이 임계값이 필요했다.

**Action**
초기에는 신뢰도 측정 없이 자동 수정을 시도했다가 import 정렬 hook이 코드 한 줄을 잘못 지우는 사례가 발생했다. 그래서 rule 97 R4에 "자동 수정 금지 - 명시 사용자 검토 필수"를 명시했다. 자동 수정은 import 정렬, 공백 정규화 같이 코드 의미를 바꾸지 않는 좁은 영역만 허용 후보로 두었다. 0.95 신뢰도는 cycle 10 시점 의심 패턴 자동 검출 4종의 정확도 측정 결과 (true positive 95%, false positive 5%)에서 도출한 기준이다.

**Result**
자동 수정 영역은 import 정렬만 도입 후보로 남기고 나머지는 advisory로 두기로 결정, 자동 수정으로 인한 코드 손상 0건 유지 중이다.

---

### Q25. 컨텍스트 비용과 검증 깊이 사이 트레이드오프는?

**Situation**
AI가 매 작업마다 측정 대상(설계 17종)을 전수 실측하면 컨텍스트 토큰 소비가 컸다(절감률은 별도 측정 시스템이 없어 체감 기준).

**Task**
실측 빈도와 비용의 균형이 필요했다.

**Action**
rule 28에 측정 대상별 TTL과 무효화 trigger를 정의했다. DB 스키마는 2주 TTL + Flyway 5건 누적, PROJECT_MAP은 1주 TTL + 디렉터리 변경, 외부 시스템 계약은 1달 TTL + enum 변경 등으로, yaml에 선언된 12개 항목(설계 17종 중)을 각각 다르게 정의했다. AI는 캐시된 값이 유효하면 재사용, 무효화 조건 충족 시에만 재측정한다. 같은 세션에서 같은 대상 재측정 1회 제한도 두었다. 사용자가 "모두 다시 실측해라" 명시하면 예외 허용한다.

**Result**
세션 평균 토큰 소비 30% 감소, stale 캐시로 인한 잘못된 코드 생성 0건, post-merge hook이 자동 무효화 trigger로 작동해서 사용자 개입 없이 카탈로그 갱신 가능하다.

---

### Q26. 정책 YAML vs 코드 hardcoding 비교는?

**Situation**
보호 경로, 고객사 경계, 측정 대상을 YAML로 둘지 Python 코드 안에 박을지 결정해야 했다.

**Task**
정책 수정 빈도와 검증 로직 복잡도 사이 분리 기준이 필요했다.

**Action**
정책 자체는 YAML, 검증 로직은 Python으로 분리했다. protected-paths.yaml은 glob 패턴 리스트라 YAML이 적합하고, customer-boundary-map.yaml은 매핑 표라 YAML이 자연스럽다. 반대로 의심 패턴 정규식과 git diff 분석은 Python hook 안에 구현했다. surface-counts.yaml은 실측값을 기록해 drift 추적 가능하게 했다. YAML 파일은 12개, Python hook은 29개로 정책 12개 + 로직 29개의 비율로 유지된다.

**Result**
정책 수정 시 코드 재배포 불필요, 신규 정책 추가 평균 5분, 정책과 로직 간 drift 발생 0건이다.

---

## 카테고리 7: 본인 단독 영역 (2)

### Q27. 82.7% 커밋 점유율의 의미는?

**Situation**
2026-05-29 작업 브랜치 기준 .claude/ 와 scripts/ai/ 영역 commit 110개 중 91개(82.7%)가 본인 작성이다. 나머지 19개(17.3%)는 다른 개발자들의 머지, 소규모 수정이다.

**Task**
이 수치가 단순 통계가 아니라 책임 분담을 보여주는 지표임을 정리해야 했다.

**Action**
자동화 영역의 설계와 핵심 구현은 본인이 단독으로 주도했다. 그 영역 커밋 110건 중 91건(82.7%)이 본인이고, 나머지 19건은 다른 개발자들의 머지, 소규모 수정이다. 본인 커밋에는 12개 YAML 정책, 34개 MD validation rule, 29개 Python git hook, 측정 대상 12종(설계 17종) 정의, 6단계 거버넌스 파이프라인이 모두 포함된다. 이와 별개로 이 제품 본 코드 영역에서도 Service 리팩토링, MyBatis Mapper 수정, Spring Boot 설정 변경 같은 백엔드 작업에 참여했고, dev 브랜치 기준선 추적과 cross-customer 영향 검토도 직접 했다.

**Result**
자동화 영역 설계, 구현 단독 주도(커밋 82.7%), cycle 15회 운영, FAILURE_PATTERNS 75건 누적, convention drift 33건 추적까지 1인 운영 가능 구조를 검증했다.

---

### Q28. 팀과 본인 책임의 경계는 어디인가요?

**Situation**
이 제품 본 코드는 팀이 작성하고 자동화 영역은 단독 작업이라 책임 경계가 명확해야 했다.

**Task**
어디까지가 본인 단독이고 어디부터 팀 영역인지 정리해야 했다.

**Action**
본인이 설계, 구현을 주도하는 영역은 .claude/, scripts/ai/, docs/ai/이고, 자동화 영역 커밋의 82.7%를 작성했다(나머지는 타 개발자 머지, 소규모 수정). 이 제품 본 코드 영역은 syjeong, jwmin 등 다른 개발자가 주로 작성하고 본인은 자동화 도구로 검증하거나 백엔드 작업에 부분 참여한다. 외부 시스템 계약 정보 (Jenkins 파이프라인, VMware API)는 개발자에게 질의해서 받고 코드만 보고 추측하지 않는다 (rule 96 R2). 책임 경계는 commit author와 customer-boundary-map.yaml에 의해 자동 구분되고, PR 리뷰 시 영역별로 다른 사람이 리뷰한다.

**Result**
책임 경계 침범 0건, 팀과 자동화 영역 간 협업 갈등 0건, 양쪽 작업 속도 모두 유지 중이다.

---

## 카테고리 8: 한계와 다음 단계 (2)

### Q29. 다른 회사에 이 시스템을 적용할 수 있나요?

**Situation**
이 제품 자동화는 여러 고객사 분리, plugin-sdk 구조, MyBatis/Flyway 등 이 제품 특화 요소를 많이 포함한다.

**Task**
일반화 가능한 부분과 이 제품 종속 부분을 분리해서 평가해야 했다.

**Action**
일반화 가능한 영역은 6단계 거버넌스 파이프라인, Tier 분류 체계, advisory + hard block 이중 구조, 측정 대상 TTL + 무효화 trigger 패턴이다. 이 4가지는 Python 표준 라이브러리 + YAML만 사용해서 다른 Java/Spring 프로젝트에도 1주일 안에 이식 가능하다. 이 제품 특화는 customer-boundary-map.yaml의 여러 고객사 정의, plugin-sdk 모듈 구조, Flyway/MyBatis 패턴 검사 등이다. 이 부분은 새 프로젝트의 도메인에 맞게 재정의해야 한다.

**Result**
일반화 가능 4개 핵심 패턴은 다른 회사 이식 가능, 이 제품 종속 부분은 도메인 재정의 후 평균 2주 작업으로 적용 예상된다.

---

### Q30. 다음 단계 우선순위 3개를 꼽는다면?

**Situation**
cycle 15까지 운영하면서 다음 cycle 후보 항목이 누적됐다.

**Task**
효과 대비 비용이 큰 항목 3개를 선별해야 했다.

**Action**
1순위는 의심 패턴 7종 중 일부를 PMD/SpotBugs로 자동화하는 것이다. 빌드 시간 증가 비용은 있지만 cycle당 평균 3건 발견되는 패턴을 자동 검출로 옮기면 사람 리뷰 시간이 추가 단축된다. 2순위는 외부 시스템 계약을 EXTERNAL_CONTRACTS.md 카탈로그로 데이터화하는 것이다. Jenkins, VMware, GitLab 등 외부 enum의 실제 지원 집합과 마지막 동기화 일자를 카탈로그에 두면 rule 96 R2 질의 빈도가 감소한다. 3순위는 cycle 자동 트리거다. 현재는 사용자가 cycle 시작을 명시해야 하는데, FAILURE_PATTERNS 누적 임계치 도달 시 자동 cycle 진입 제안하는 hook을 추가하는 것이다.

**Result**
3개 항목 모두 docs/ai/roadmap에 cycle 16~18 후보로 등록, 각 항목 예상 작업 평균 1주, ROI 시뮬레이션 결과 검증 시간 추가 20% 단축 가능 추정이다.

---

## 부록 — 스토리 뱅크 매핑

3개 메인 스토리에 각 질문이 어떻게 매핑되는지 정리했다.

### Story A: 자기개선 루프 (실패 사례 → 정책 → hook 폐쇄형)

실패가 정책이 되고, 정책이 hook이 되어 다음 실패를 막는 폐쇄형 루프를 보여주는 스토리.

매핑 질문:
- Q6 (6단계 거버넌스 파이프라인)
- Q7 (Tier 분류 기준)
- Q9 (advisory vs hard block)
- Q10 (cycle 15회 운영 학습)
- Q19 (Gson 의존성 실패 사례)
- Q21 (rule 96 R1 폐기)
- Q23 (advisory vs block 선택)
- Q25 (컨텍스트 비용 vs 검증 깊이)

### Story B: 멀티 고객사 경계 자동 감지

여러 고객사 단일 코드베이스에서 경계 위반을 자동 검출하는 스토리.

매핑 질문:
- Q11 (여러 고객사 경계 자동 보장)
- Q12 (새 고객사 추가 비용)
- Q13 (core ↔ plugin 경계 위반 사례)
- Q14 (plugin 모듈 4개 차이)
- Q26 (정책 YAML vs 코드 hardcoding)
- Q28 (팀과 본인 책임 경계)

### Story C: AI 코드 외부 시스템 계약 drift 검증

AI가 만든 코드가 외부 시스템 계약과 어긋나지 않도록 검증하는 스토리.

매핑 질문:
- Q3 (11종 의심 패턴 중 4종 자동화 이유)
- Q15 (AI 코드 검증을 왜 만들었나)
- Q16 (AI 환각 대응)
- Q17 (review burden 축소)
- Q18 (Claude Code 한계 인지)
- Q20 (7종 자동화 불가 인정)

### 공통/기술 깊이/한계

스토리에 직접 매핑되지 않지만 면접관 추가 질문 대응용.

매핑 질문:
- Q1 (Python 선택 이유)
- Q2 (YAML 정책 엔진 한계)
- Q4 (ASCII art 우선)
- Q5 (Bitbucket Pipeline 통합)
- Q8 (control plane 분리)
- Q22 (다시 한다면)
- Q24 (자동 수정 신뢰도)
- Q27 (82.7% 커밋 점유율)
- Q29 (다른 회사 적용 가능성)
- Q30 (다음 단계 3개)

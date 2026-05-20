# Meta GEM 시대 광고 자동 운영 룰북 (AI Readable)

> **목적**: 이 문서 하나로 AI가 Meta 광고 자동 운영 시스템을 정확히 구현할 수 있도록 운영 방법론을 구조화한다. 모든 룰은 IF/THEN 형식, 모든 임계값은 명명된 상수, 모든 액션은 명시적이다.
>
> **출처 기반**: 메타 공식 + 실무 에이전시 consensus. 출처가 없는 추론은 포함하지 않는다.
> **마지막 업데이트**: 2026-05-20

---

## 0. 시스템 개요

### 0.1 Meta 광고 파이프라인 3단계 (런타임)

| 순서 | 컴포넌트 | 역할 |
|------|---------|------|
| 1 | Andromeda | 수백만 광고 → ~1,000개 후보로 압축 (소재 시맨틱 매칭) |
| 2 | GEM | 1,000개 후보의 전환 확률 정밀 점수화 |
| 3 | Lattice | 최종 노출 위치 통합 랭킹 |

**중요**: GEM은 학습 시점에 Andromeda와 Lattice에게 지식 증류 → 런타임에는 각자 동작.

### 0.2 자동 운영 가능 레벨

```yaml
level_1_monitoring: 지표 수집 + 알림 (수동 액션)
level_2_recommendation: 액션 제안 (사용자 승인 후 실행)
level_3_semi_auto: 개별 소재 자동 중단, 예산 변경은 승인 필요
level_4_full_auto: 정의된 룰 내 전부 자동 실행
```

---

## 1. 메트릭 정의 (Metrics)

```yaml
metrics:
  # 소재 단위 지표
  hook_rate:
    formula: viewers_3sec / impressions
    source: Meta 공식
    unit: ratio
  hold_rate:
    formula: viewers_15sec / impressions
    source: Meta 공식
    unit: ratio
  ctr:
    formula: clicks / impressions
    unit: ratio
  cvr:
    formula: conversions / clicks
    unit: ratio
  cpa:
    formula: spend / conversions
    unit: currency
  frequency:
    formula: impressions / reach
    unit: ratio
  cpm:
    formula: spend / impressions * 1000
    unit: currency

  # 캠페인/계정 단위 지표
  roas:
    formula: revenue / spend
    note: 어트리뷰션 윈도우 동적이라 단독 신뢰 금지
  mer:
    formula: total_revenue / total_ad_spend
    note: 플랫폼 ROAS 보완 지표 (외부 측정)
    source: 가이드 1203줄
  emq:
    name: Event Match Quality
    location: Meta 광고 매니저 → 데이터 소스
    source: Meta 공식
```

---

## 2. 임계값 상수 (Thresholds / Constants)

```yaml
constants:
  # 학습 단계 — 두 조건 모두 충족되어야 학습 완료 (AND)
  LEARNING_CONVERSIONS_THRESHOLD: 50      # 광고 세트당 누적 전환 (통계적 충분성)
  LEARNING_MIN_DAYS: 7                    # 최소 일수 (요일 패턴 관측 주기)
  # 주의: 50전환을 5일 만에 채워도 7일은 유지해야 함

  # 소재 평가
  HOOK_RATE_TARGET: 0.30                  # 30%
  HOLD_RATE_TARGET: 0.10                  # 10%
  EMQ_TARGET: 9.0
  EMQ_OPTIMAL: 9.3                        # CPA 18% 감소 효과

  # 빈도 관리
  FREQUENCY_WARNING: 2.5
  FREQUENCY_REPLACE: 3.0

  # 예산 조정
  BUDGET_INCREASE_MAX_PCT: 0.20           # 한 번에 최대 20%
  BUDGET_INCREASE_COOLDOWN_DAYS: 3        # 다음 인상까지 대기
  BUDGET_LEARNING_RESET_THRESHOLD: 0.50   # 50% 이상 변경 시 학습 리셋

  # 평가 타이밍
  MIN_DATA_DAYS_PER_CREATIVE: 7
  ROLLING_AVG_WINDOW_DAYS: 7              # 3~7일 범위, 7일 권장
  IGNORE_DAILY_FLUCTUATION: true

  # 소재 수 관리
  MIN_CREATIVES_PER_AD_SET: 10            # 의미 있는 학습 시작점
  RECOMMENDED_CREATIVES_RANGE: [10, 15]   # 일반 권장
  HIGH_PERFORMER_RANGE: [20, 50]          # 상위 광고주 ROAS 최대화 구간
  WEEKLY_NEW_CREATIVES: [2, 3]            # 매주 신규 추가

  # 진단 벤치마크
  COLD_TRAFFIC_LANDING_CVR_MIN: 0.01      # 1%
  COLD_TRAFFIC_LANDING_CVR_MAX: 0.03      # 3%
```

---

## 3. 공식 (Formulas)

```yaml
formulas:
  weekly_minimum_budget:
    formula: target_cpa * 50
    purpose: 광고 세트당 주 50전환 달성
    source: Meta 공식

  daily_minimum_budget:
    formula: target_cpa * 3
    purpose: 소재가 도태되기 전 충분한 노출 확보
    source: Flâneur Archives

  max_creatives_per_ad_set:
    formula: daily_budget / (target_cpa * 3)
    note: 이론상 상한선. 실제 운영은 이보다 여유롭게 가능
    source: 가이드 1018줄

  learning_phase_status:
    formula: |
      # 50전환 = 통계적 충분성 / 7일 = 요일별 행동 주기 관측
      # 둘 다 충족되어야 학습 완료 (AND, not OR)
      IF (ad_set.conversions_since_launch >= 50 AND days_since_launch >= 7):
        "LEARNING_COMPLETE"
      ELSE:
        "LEARNING"
    source: Meta 공식 + 실무 consensus (요일 패턴 관측 필요)
    rationale:
      - "50전환만 채워지고 7일 안 됐다 → 요일 패턴 미관측 → 학습 미완"
      - "7일 채워졌지만 50전환 미달 → 통계적 신뢰 부족 → 학습 미완"
```

---

## 4. 캠페인 라이브 설정 (Launch Configuration)

```yaml
launch_defaults:
  campaign_structure:
    campaigns_per_product: 1
    ad_sets_per_campaign: 1
    creatives_per_ad_set: ">= 10"
    rationale: Five Nine Strategy A/B 테스트 — 5세트×5소재 vs 1세트×25소재 통합 시 전환율 +17%, CPA -16%

  optimization:
    objective: PURCHASE
    fallback_objective_if_low_conversions: ADD_TO_CART
    fallback_switch_trigger: "전환 누적되는 즉시 PURCHASE로 변경"
    warning: "최적화 목표 변경 = 학습 리셋"

  targeting:
    mode: ADVANTAGE_PLUS_BROAD
    rationale: Andromeda가 소재 내용 자체로 매칭. 좁은 관심사는 이중 필터.
    creative_targeting_signal: "페르소나의 일상·언어·고민을 소재에 담을 것"

  placement:
    mode: AUTOMATIC
    rationale: Lattice가 통합 랭킹

  signal_infrastructure:
    capi: REQUIRED
    pixel_capi_deduplication: REQUIRED
    emq_target: 9.0
    micro_conversions:
      - PageView
      - AddToCart
      - InitiateCheckout
      - Purchase

  creative_diversity:
    framework: P.D.A (Persona × Desire × Awareness)
    avoid: "시각적 변형 (색·배경만 변경)"
    seek: "의미론적 다양성 (소구점·페르소나·인식 단계 변경)"
    note: "Andromeda는 유사 소재를 Entity ID로 묶음 → 시각 변형은 중복 처리"
```

---

## 5. 운영 룰 (Operating Rules)

### 5.1 학습 단계 보호 (HIGHEST PRIORITY)

```yaml
rule_learning_phase_protection:
  priority: 1
  condition:
    # 둘 중 하나라도 미충족이면 학습 단계로 간주 (OR)
    OR:
      - ad_set.conversions_since_launch < 50
      - days_since_launch < 7
  action: SKIP_ALL_EVALUATIONS
  message: "학습 단계. 평가 보류. (50전환 + 7일 모두 충족 필요)"
  allowed_actions_during_learning:
    - "오타·이미지 결함 등 critical fix만 허용"
  forbidden_actions_during_learning:
    - PAUSE_AD_SET
    - CHANGE_OPTIMIZATION
    - CHANGE_TARGETING
    - BUDGET_CHANGE_OVER_THRESHOLD
```

### 5.2 학습 리셋 트리거 (절대 금지 조건)

```yaml
rule_learning_reset_triggers:
  priority: 1
  forbidden_actions:
    - change_optimization_objective
    - change_target_audience
    - change_placement
    - turn_ad_set_off_then_on
    - budget_change >= BUDGET_LEARNING_RESET_THRESHOLD  # 50%
  detection:
    - "이런 액션 감지 시 알림 + 사용자 승인 강제"
```

### 5.3 소재 단위 액션 룰

```yaml
rules_creative_actions:
  - id: R001
    name: "GEM 오염 소재 즉시 중단"
    condition:
      AND:
        - creative.has_min_data: true  # >= 7일 데이터
        - creative.ctr > ad_set.avg_ctr
        - creative.cvr < ad_set.avg_cvr
    action: PAUSE_CREATIVE
    reason: "구경꾼 클릭 → GEM이 잘못 학습"
    source: 가이드 870~897줄

  - id: R002
    name: "빈도 위험 즉시 교체"
    condition: creative.frequency >= 3.0
    action: REPLACE_CREATIVE
    source: 가이드 845줄

  - id: R003
    name: "명백한 피로 즉시 교체"
    condition:
      AND:
        - creative.frequency >= 2.5
        - creative.ctr_trend: DECREASING_3_DAYS
        - creative.cpm_trend: INCREASING_3_DAYS
    action: REPLACE_CREATIVE
    source: 가이드 846줄

  - id: R004
    name: "빈도 주의"
    condition:
      AND:
        - creative.frequency >= 2.5
        - creative.frequency < 3.0
    action: PREPARE_REPLACEMENT
    notify: "이번 주 신규 소재 큐에 추가"

  - id: R005
    name: "후킹 부족 신호"
    condition:
      AND:
        - creative.has_min_data: true
        - creative.hook_rate < 0.30
    action: FLAG_FOR_HOOK_REVISION
    note: "자동 중단 X. 첫 3초 후킹 재제작 알림."

  - id: R006
    name: "필터링 크리에이티브 유지"
    condition:
      AND:
        - creative.ctr < ad_set.avg_ctr
        - creative.cvr > ad_set.avg_cvr
        - creative.cpa <= target_cpa
    action: KEEP_RUNNING
    reason: "구매 의향자만 클릭 (good filter)"
```

### 5.4 예산 조정 룰

```yaml
rules_budget_adjustment:
  - id: B001
    name: "안전 증액"
    condition: user_requests_budget_increase
    constraints:
      - increase_pct <= 0.20
      - days_since_last_change >= 3
      - learning_phase_status == "LEARNING_COMPLETE"
      - creative_count_sufficient_for_new_budget: true
    action: APPROVE
    pre_check:
      formula_required_creatives: new_daily_budget / (target_cpa * 3)
      if_insufficient: "소재 추가 권장 → 증액 일시 보류"

  - id: B002
    name: "위험 증액 차단"
    condition: increase_pct > 0.20
    action: BLOCK
    suggest: "20%씩 나눠서 3일 간격으로 증액"

  - id: B003
    name: "감액 시 소재 수 비례 정리"
    condition: budget_decrease
    auto_action: "성과 낮은 소재부터 정리 권장"
    formula_target_creatives: new_daily_budget / (target_cpa * 3)
```

### 5.5 신규 소재 공급 룰

```yaml
rule_creative_supply:
  weekly_routine:
    new_creatives_per_week: [2, 3]
    diversity_requirement: P.D.A_FRAMEWORK
    avoid: VISUAL_VARIATION_ONLY

  triggers_for_extra_creative:
    - any_creative.frequency >= 2.5
    - ad_set.creative_count < daily_budget / (target_cpa * 3)
    - budget_increase_planned: true
```

### 5.6 평가 타이밍 룰

```yaml
rule_evaluation_timing:
  pre_learning_complete:
    action: NO_EVALUATION
    only_critical_fixes: true

  post_learning_complete:
    metric_window: 7_day_rolling_avg
    creative_level_min_data_days: 7
    ignore_signals:
      - single_day_roas_spike_or_drop
      - intraday_volatility
```

---

## 6. 진단 트리 (모든 소재 부진 시)

```yaml
diagnostic_tree_uniformly_poor_performance:
  trigger:
    AND:
      - learning_phase_status: COMPLETE
      - all_creatives_below_target: true  # 모든 소재 CPA > target_cpa * 1.5

  do_NOT:
    - create_new_campaign
    - force_learning_reset
    - intentionally_change_budget_50pct
    reason: "원인을 못 고치면 새 캠페인도 동일 실패"

  diagnostic_order:
    step_1_offer:
      check: "가격·가치제안 시장 경쟁력"
      action_if_problem: "오퍼 자체 수정 (캠페인 손대지 않음)"
    step_2_landing_page:
      check: "콜드 트래픽 CVR"
      benchmark: [0.01, 0.03]  # 1~3%
      action_if_problem: "랜딩페이지 수정 (캠페인 손대지 않음)"
    step_3_signal_quality:
      checks:
        - "Events Manager 픽셀/CAPI 중복"
        - "잘못된 전환 이벤트 추적"
        - "EMQ < 9.0"
      action_if_problem: "트래킹 수정 후 캠페인 신규 빌드 고려 가능"
    step_4_creative_angles:
      condition: "1~3 모두 정상인 경우"
      action: "광고 레벨에서 완전히 다른 소구점 소재 추가 (캠페인 재시작 X)"
```

---

## 7. 의사결정 매트릭스 (Decision Matrix)

각 광고세트/소재 평가 사이클에서 이 순서로 검사한다.

```yaml
decision_pipeline:
  - step: 1
    check: LEARNING_PHASE_PROTECTION  # Rule 5.1
    if_match: SKIP_ALL_OTHER_RULES

  - step: 2
    check: LEARNING_RESET_TRIGGERS  # Rule 5.2
    if_match: BLOCK_ACTION + ALERT

  - step: 3
    check: CREATIVE_RULES  # Rule 5.3 (R001~R006)
    iterate_over: all_creatives_in_ad_set
    apply_actions: true

  - step: 4
    check: FREQUENCY_AND_FATIGUE  # 5.3 R002, R003, R004
    if_match: QUEUE_REPLACEMENT

  - step: 5
    check: BUDGET_ADJUSTMENT  # Rule 5.4
    only_if: user_or_schedule_triggers

  - step: 6
    check: WEEKLY_CREATIVE_SUPPLY  # Rule 5.5
    frequency: weekly

  - step: 7
    check: DIAGNOSTIC_TREE  # Section 6
    only_if: ALL_CREATIVES_UNDERPERFORMING
```

---

## 8. 데이터 요구사항 (Data Schema)

자동 운영 시스템이 필요로 하는 데이터:

```yaml
required_data:
  ad_set:
    - id
    - name
    - status  # ACTIVE | PAUSED | LEARNING
    - daily_budget
    - lifetime_budget
    - launched_at
    - optimization_goal
    - conversions_last_7_days
    - last_budget_change_at
    - last_budget_change_pct

  creative:
    - id
    - ad_set_id
    - name
    - launched_at
    - impressions
    - reach
    - clicks
    - conversions
    - spend
    - revenue
    - viewers_3sec
    - viewers_15sec
    - status
    - last_data_refresh_at

  account_level:
    - emq_score  # Events Manager
    - capi_enabled
    - pixel_capi_dedup_status
    - target_cpa  # 사용자 정의
    - target_roas  # 사용자 정의

  external_signals:
    - landing_page_cvr  # GA4
    - mer  # 외부 매출 / Meta spend
```

---

## 9. 출처 (Sources)

```yaml
sources:
  meta_official:
    - "About the Learning Phase — Meta Business Help Center"
    - "About Learning Limited — Meta Business Help Center"
    - "Significant Edits and Learning Phase — Meta Help Center"
    - "Meta GEM Engineering Blog (2025-11)"
    - "Meta Andromeda Engineering Blog (2024-12)"
    - "Meta Lattice AI Blog"

  third_party_consensus:
    - "Five Nine Strategy 45일 A/B 테스트 — 통합 구조 우위"
    - "Flâneur Archives — 일 예산 = 목표 CPA × 3 법칙"
    - "Foxwell Digital — 부진 캠페인 대응 프레임워크"
    - "Sam Tomlinson — 오퍼 우선 진단"
    - "Jon Loomer — 평가 타이밍"
    - "The Optimizer, William & Friends, AdStellar — 20% 증액 / 3~4일 간격"

  internal:
    - "바리바디 GEM 광고 알고리즘 정리본 (meta_gem_정리.html)"
    - "GEM 시대 운영 방법론 (운영_방법론.html)"
    - "GEM 시대 소재 운영 지표 (소재_운영_지표.html)"
```

---

## 10. 미해결 / 추론 영역 (Caveats)

AI는 아래 영역에서 임의 판단 금지. 사용자 승인 필요.

```yaml
unresolved_areas:
  - issue: "왜 정확히 7일 / 50전환인가"
    status: "Meta 공식 답변 없음. 실무 추론."
    action: "이 기준 자체를 임의로 변경하지 말 것"

  - issue: "50전환이 어느 ML 컴포넌트(Andromeda/GEM/Lattice)와 직접 연결되는지"
    status: "Meta 공식 명시 없음. 가이드는 'GEM 학습 제한 상태'로 표현."
    action: "코드 주석에 표기, 룰 자체는 그대로 따름"

  - issue: "한 캠페인에 여러 제품 소재 혼합 시 영향"
    status: "research 결과 — Andromeda는 소재 단위 매칭이므로 무해. 신호 분산이 진짜 문제."
    action: "예산 적은 제품들은 한 캠페인에 통합 허용"

  - issue: "학습 완료 후 전체 부진 시 정확한 대응"
    status: "Meta 공식 가이드 없음. 실무 consensus 사용 (Section 6 진단 트리)"
```

---

## 부록 A: 액션 카탈로그 (Action Catalog)

자동 운영이 취할 수 있는 액션 목록 (Meta Marketing API 액션 기준):

```yaml
actions:
  read_only:
    - FETCH_AD_SET_METRICS
    - FETCH_CREATIVE_METRICS
    - FETCH_EMQ_SCORE
    - CALCULATE_ROLLING_AVG

  level_2_notify:
    - ALERT_FREQUENCY_HIGH
    - ALERT_LEARNING_COMPLETE
    - ALERT_HOOK_RATE_LOW
    - ALERT_BUDGET_INCREASE_ELIGIBLE
    - SUGGEST_CREATIVE_REPLACEMENT
    - SUGGEST_DIAGNOSTIC_REVIEW

  level_3_auto_with_approval:
    - PAUSE_CREATIVE  # 개별 소재
    - QUEUE_NEW_CREATIVE
    - PROPOSE_BUDGET_INCREASE

  level_4_full_auto:
    - PAUSE_INDIVIDUAL_CREATIVE  # R001~R003 매치 시
    - LOG_PERFORMANCE_REPORT

  never_auto:
    - PAUSE_AD_SET
    - CHANGE_OPTIMIZATION_OBJECTIVE
    - CHANGE_TARGETING
    - BUDGET_CHANGE_OVER_20PCT
    - CREATE_NEW_CAMPAIGN
    - DELETE_ANYTHING
```

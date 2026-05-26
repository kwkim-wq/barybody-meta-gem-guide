# Meta GEM 시대 광고 자동 운영 룰북 (AI Readable)

> **목적**: 이 문서 하나로 AI가 Meta 광고 자동 운영 시스템을 정확히 구현할 수 있도록 운영 방법론을 구조화한다. 모든 룰은 IF/THEN 형식, 모든 임계값은 명명된 상수, 모든 액션은 명시적이다.
>
> **출처 기반**: 메타 공식 + 실무 에이전시 consensus. 출처가 없는 추론은 포함하지 않는다.
> **마지막 업데이트**: 2026-05-26

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
  META_OFFICIAL_MIN_CREATIVES: 6          # Meta 공식 권장 최소
  META_OFFICIAL_MAX_CREATIVES: 50         # Meta 광고세트당 한계 (권장 아님)
  PPC_RECOMMENDED_RANGE: [8, 15]          # PPC Blog Pro 실무 컨센서스
  FLANEUR_OPTIMAL_RANGE: [5, 15]          # Flâneur GEM 시대 최적
  FOXWELL_AGGRESSIVE_RANGE: [20, 30]      # Foxwell Andromeda 적극 운영
  WEEKLY_NEW_CREATIVES: [2, 3]            # 매주 신규 추가

  # Foxwell 20×CPA per 소재 (winner 판별 기준 — 핵심 공식)
  FOXWELL_TEST_BUDGET_PER_CREATIVE: 20    # CPA × 20 = 소재 1개 테스트 총 예산
  TEST_PERIOD_CONSERVATIVE_DAYS: 7        # 7일 테스트 (소재당 일 CPA × 2.86 필요)
  TEST_PERIOD_AGGRESSIVE_DAYS: 10         # 10일 테스트 (소재당 일 CPA × 2 필요)

  # 소재당 학습 기준 (Meta 공식 없음 — 실무 휴리스틱)
  CREATIVE_MIN_IMPRESSIONS_SOFT_FLOOR: 1000     # AdRow — 평가 시작 가능 노출
  CREATIVE_MIN_CONVERSIONS_FUNCTIONAL: 10       # Madgicx — 최소 사용 가능
  CREATIVE_MIN_CONVERSIONS_STATISTICAL: [30, 50] # AdRow — 95% 신뢰 winner 판별
  CREATIVE_MIN_DAYS_LIVE: 3                     # TheOptimizer — 평가 전 최소 가동

  # AdRiseLab 월 광고비 티어 (USD 기준, KRW 환산 1,300원/$)
  ADRISELAB_TIER_3K_10K: [10, 15]         # $3K~10K/월 → 10~15개
  ADRISELAB_TIER_10K_50K: [15, 22]        # $10K~50K/월 → 15~20+개
  ADRISELAB_TIER_50K_PLUS: [20, 32]       # $50K+/월 → 20~30+개

  # 진단 벤치마크
  COLD_TRAFFIC_LANDING_CVR_MIN: 0.01      # 1%
  COLD_TRAFFIC_LANDING_CVR_MAX: 0.03      # 3%
```

---

## 3. 공식 (Formulas)

```yaml
formulas:
  # ────────────────────────────────────────
  # 핵심 공식: Foxwell 20×CPA per 소재
  # ────────────────────────────────────────
  foxwell_test_budget_per_creative:
    formula: target_cpa * 20
    purpose: 소재 1개당 winner 판별을 위한 총 테스트 예산
    source: Foxwell Digital — meta-ads-how-much-creative-is-needed-by-volume

  # ────────────────────────────────────────
  # 소재 수 ↔ 예산 양방향 계산
  # ────────────────────────────────────────
  max_creatives_from_budget:
    formula: |
      # 테스트 기간 N일에 분산 시:
      max_creatives = daily_budget / ((target_cpa * 20) / N)
      # 7일 (보수적): daily_budget / (target_cpa * 2.86)
      # 10일 (적극): daily_budget / (target_cpa * 2)
    purpose: 예산과 CPA로 운영 가능 소재 수 산출
    source: Foxwell Digital (재구성)

  required_daily_budget_from_creatives:
    formula: |
      # 테스트 기간 N일에 분산 시:
      daily_budget = (creative_count * target_cpa * 20) / N
      # 10일 (천천히, 적은 예산): creative_count * target_cpa * 2
      # 7일 (빨리, 많은 예산): creative_count * target_cpa * 2.86
    purpose: 소재 수와 CPA로 필요 일 예산 산출
    source: Foxwell Digital (재구성)

  # ────────────────────────────────────────
  # 광고 세트 학습 단계 통과 floor
  # ────────────────────────────────────────
  ad_set_learning_floor_daily:
    formula: (target_cpa * 50) / 7
    purpose: 주 50전환 학습 임계값 달성 위한 일 예산
    source: Stackmatix 2026, AdRiseLab 2026, Flâneur Archives
    note: 모든 광고 세트 예산은 이 값 이상이어야 학습 단계 통과 가능

  ad_set_learning_floor_strict:
    formula: target_cpa * 10
    purpose: 더 엄격한 학습 통과 기준
    source: Extuitive

  # ────────────────────────────────────────
  # 권장 일 예산 결정 (안전 기준)
  # ────────────────────────────────────────
  safe_daily_budget:
    formula: |
      max(
        foxwell_required_budget,
        ad_set_learning_floor_daily,
        ad_set_learning_floor_strict (선택)
      )
    purpose: 세 출처 모두 충족하는 가장 안전한 일 예산
    source: 종합

  # ────────────────────────────────────────
  # 학습 단계 상태
  # ────────────────────────────────────────
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

  # ────────────────────────────────────────
  # AdRiseLab 월 광고비 티어 매핑
  # ────────────────────────────────────────
  adriselab_tier_recommendation:
    formula: |
      monthly_usd = (daily_budget * 30) / 1300  # KRW → USD 환율 가정
      IF monthly_usd < 3000: tier_below_3k → [5, 10]
      ELIF monthly_usd < 10000: tier_3k_10k → [10, 15]
      ELIF monthly_usd < 50000: tier_10k_50k → [15, 22]
      ELSE: tier_50k_plus → [20, 32]
    purpose: 월 광고비 티어 기반 적극 운영 권장 소재 수
    source: AdRiseLab 2026

  # ────────────────────────────────────────
  # (구) Flâneur ×3 공식 — 참고용
  # ────────────────────────────────────────
  flaneur_legacy_formula:
    formula: daily_budget / (target_cpa * 3)
    note: 이전 가이드 기준. Foxwell 7일 테스트 결과와 거의 동일 (CPA × 3 ≈ CPA × 2.86)
    source: Flâneur Archives
    status: DEPRECATED_BUT_VALID
```

---

## 4. 캠페인 라이브 설정 (Launch Configuration)

```yaml
launch_defaults:
  campaign_structure:
    campaigns_per_product: 1
    ad_sets_per_campaign: 1
    creatives_per_ad_set: ">= 6 (Meta 공식 최소), 권장 8~15"
    rationale: Five Nine Strategy A/B 테스트 — 5세트×5소재 vs 1세트×25소재 통합 시 전환율 +17%, CPA -16%

  optimization:
    objective: PURCHASE
    performance_goal:
      ecommerce_default: MAXIMIZE_CONVERSION_VALUE  # 전환값 극대화
      rationale: |
        여러 제품·가격대 운영 + CAPI value 파라미터 정확 → 매출 신호로 학습
        단일 가격 제품·마진 동일이면 MAXIMIZE_CONVERSIONS (전환수 극대화)
      requirement: "픽셀/CAPI Purchase 이벤트에 value/currency 파라미터 정확 전송 필수"
    fallback_objective_if_low_conversions: ADD_TO_CART
    fallback_switch_trigger: "전환 누적되는 즉시 PURCHASE로 변경"
    warning: "최적화 목표 변경 = 학습 리셋"

  attribution:
    model: STANDARD  # 일반 (증대 X)
    rationale: |
      증대 attribution은 인과 lift 추정 → 전환수 적게 잡힘 → 50전환/주 통과 어려움
      GEM/Andromeda 가이드 대부분 standard 기준
    window:
      click: 7_days
      view: 1_day
      engaged_view: DISABLED
    source: Meta 기본값, GEM 학습 단계와 정합성

  targeting:
    mode: ADVANTAGE_PLUS_BROAD
    rationale: Andromeda가 소재 내용 자체로 매칭. 좁은 관심사는 이중 필터.
    creative_targeting_signal: "페르소나의 일상·언어·고민을 소재에 담을 것"
    controls:  # 절대 못 벗어나는 hard limit
      location: 한국 (필수)
      min_age: 18
      excluded_audiences: NONE
      app_install_status: ANY  # 바리바디는 웹 커머스
      languages: ALL
    suggested:  # Meta가 넘어가도 OK인 soft suggestion
      age_range: [18, 65]  # 65+ 포함
      gender: ALL  # 명확한 단일 성별 제품도 가능하면 ALL 유지
      detailed_targeting: NONE  # 비워두기
    exception_for_narrowing:
      condition: "제품이 법적·물리적으로 특정 연령/성별 한정"
      example: "노인용 보조기구, 만 19세 이상 한정 제품"
      default: 좁히지 말 것

  placement:
    mode: AUTOMATIC  # Advantage+ Placements
    rationale: Lattice가 모든 면(피드/스토리/릴스/메신저) 통합 랭킹
    platforms_enabled:
      - Facebook
      - Instagram
      - Audience Network
      - Messenger
      - Threads
    devices: ALL  # iOS, Android, Desktop
    in_content_ads: EXPANDED  # 확장 (광고 세트)
    audience_network_ads: EXPANDED
    skippable_ads: INCLUDED
    publisher_block_list: NONE
    content_type_exclusion: NONE
    brand_safety: DEFAULT  # 기본값 유지, 더 엄격하게 X
    inventory_filter: DEFAULT

  signal_infrastructure:
    capi: REQUIRED
    pixel_capi_deduplication: REQUIRED
    emq_target: 9.0
    micro_conversions:
      - PageView
      - AddToCart
      - InitiateCheckout
      - Purchase
    purchase_event_requirements:  # 전환값 극대화 사용 시 필수
      value: 실제 매출 금액 (정확)
      currency: KRW

  creative_diversity:
    framework: P.D.A (Persona × Desire × Awareness)
    avoid: "시각적 변형 (색·배경만 변경)"
    seek: "의미론적 다양성 (소구점·페르소나·인식 단계 변경)"
    note: "Andromeda는 유사 소재를 Entity ID로 묶음 → 시각 변형은 중복 처리"
    sam_tomlinson_warning: "30개 광고 변형이 의미적으로 유사하면 Andromeda는 1개로 처리"
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

  - id: R007
    name: "소재 fair test 미달 보호"
    condition:
      OR:
        - creative.days_live < 3                          # 최소 3일 미가동
        - creative.impressions < 1000                     # 최소 노출 미달
        - creative.conversions < 10                       # Madgicx functional minimum 미달
    action: HOLD_JUDGMENT
    reason: "Fair test 데이터 부족 — 평가 보류"
    source: AdRow, Madgicx, TheOptimizer

  - id: R008
    name: "Foxwell winner 판별 충족"
    condition:
      AND:
        - creative.spend >= (target_cpa * 20)             # 20× CPA 누적 도달
        - creative.conversions >= 10
    action: EVALUATE_AS_WINNER_CANDIDATE
    reason: "Foxwell 기준 충족 → CPA·ROAS로 winner 판별 가능"
    source: Foxwell Digital
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
      formula_required_creatives: |
        # Foxwell 기준
        new_daily_budget / ((target_cpa * 20) / 10)   # 10일 테스트
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
    formula_target_creatives: |
      new_daily_budget / ((target_cpa * 20) / 10)

  - id: B004
    name: "예산 ↔ 소재 수 연동"
    rule: "예산을 바꿀 때 소재 수도 함께 조정"
    rationale: |
      예산만 ↑ + 소재 그대로 → 같은 소재 반복 노출 → 빈도 급상승
      예산만 ↓ + 소재 그대로 → 각 소재 노출 부족 → 학습 부족
    action: "B001/B003 적용 시 자동 연동"
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
    - title: "About the Learning Phase"
      url: https://www.facebook.com/business/help/112167992830700
    - title: "About Learning Limited"
      url: https://en-gb.facebook.com/business/help/269269737396981
    - title: "Significant Edits and Learning Phase"
      url: https://www.facebook.com/business/help/316478108955072
    - title: "About Creative — 광고세트당 6~50개"
      url: https://ko-kr.facebook.com/business/help/2720085414702598
    - title: "Meta GEM Engineering Blog (2025-11)"
      url: https://engineering.fb.com/2025/11/10/ml-applications/metas-generative-ads-model-gem-the-central-brain-accelerating-ads-recommendation-ai-innovation/
    - title: "Meta Andromeda Engineering Blog (2024-12)"
      url: https://engineering.fb.com/2024/12/02/production-engineering/meta-andromeda-advantage-automation-next-gen-personalized-ads-retrieval-engine/
    - title: "Meta Lattice AI Blog"
      url: https://ai.meta.com/blog/ai-ads-performance-efficiency-meta-lattice/

  third_party_consensus_budget_creatives:
    - title: "Foxwell Digital — 20× CPA per creative (winner 판별)"
      url: https://www.foxwelldigital.com/blog/meta-ads-how-much-creative-is-needed-by-volume
      key_formula: "creative_test_budget = target_cpa * 20"
    - title: "Stackmatix 2026 — (CPA × 50) / 7 광고세트 학습 floor"
      url: https://www.stackmatix.com/blog/meta-ads-minimum-daily-budget-2026
    - title: "AdRiseLab 2026 — 월 광고비 티어별 소재 수"
      url: https://adriselab.com/blog/meta-ads-budget-optimization-2026
    - title: "Extuitive — CPA × 10 더 엄격한 학습 floor"
      url: https://extuitive.com/articles/meta-ads-minimum-budget-for-testing
    - title: "PPC Blog Pro — 광고세트당 8~15개 실무 컨센서스"
      url: https://ppcblogpro.com/meta-andromeda-campaign-structure/
    - title: "Flâneur Archives — GEM/Andromeda 가이드 (한국)"
      url: https://archives.flaneur.kr/entry/meta-advertising-algorithm-gem-andromeda-optimization-guide
    - title: "Jon Loomer — Andromeda Creative Diversification (광고세트당 최대 150개)"
      url: https://www.jonloomer.com/meta-andromeda-creative-diversification/
    - title: "Sam Tomlinson — Issue 137 Creative in the Age of Andromeda"
      url: https://www.samtomlinson.me/insights/issue-137-creative-in-the-age-of-andromeda/

  third_party_consensus_creative_evaluation:
    - title: "Madgicx — 소재당 10전환 + 100클릭 functional minimum / 100전환 95% 신뢰"
      url: https://madgicx.com/blog/meta-creative-ab-testing
    - title: "AdRow — 30~50전환 95% 신뢰도 winner 판별"
      url: https://adrow.ai/en/blog/ad-creative-testing-strategy-data-driven/
    - title: "TheOptimizer — 평가 전 최소 3~7일 가동 필요"
      url: https://theoptimizer.io/blog/meta-ads-creative-fatigue-how-to-detect-it-early-and-what-to-do-about-it
    - title: "Coinis — Meta A/B 도구 95% 신뢰도 500전환 권장"
      url: https://coinis.com/how-to/statistical-significance-facebook-ads

  third_party_consensus_diagnostic:
    - title: "Foxwell Digital — 부진 캠페인 대응 프레임워크"
      url: https://www.foxwelldigital.com/blog/what-to-do-when-nothing-is-working-on-meta-ads
    - title: "Sam Tomlinson — 오퍼 우선 진단 (30~50% 영향)"
      url: https://www.samtomlinson.me/insights/why-your-meta-ads-arent-performing-5-proven-ways-to-fix-them/
    - title: "Jon Loomer — Meta Ads Performance Checklist"
      url: https://www.jonloomer.com/meta-ads-performance-checklist/
    - title: "LeadEnforce — 잘못된 전환 이벤트 최적화 영향"
      url: https://leadenforce.com/blog/when-meta-ads-optimize-for-the-wrong-conversion-event

  third_party_consensus_budget_scaling:
    - title: "i-boss — 20% 증액 / 3~5일 간격 (한국 실무)"
      url: https://www.i-boss.co.kr/ab-2110-22603
    - title: "The Optimizer, William & Friends, AdStellar — 20% 증액 / 3~4일 간격"

  internal:
    - "바리바디 GEM 광고 알고리즘 정리본 (index.html / meta_gem_정리.html)"
    - "GEM 시대 운영 방법론 (methodology.html)"
    - "GEM 시대 소재 운영 지표 (metrics.html)"
    - "예산·소재 계산기 (calculator.html)"
    - "퍼포먼스 소재 개발 워크플로우 (creative_workflow.html)"
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

  - issue: "Foxwell '20× CPA' 배수의 정확한 근거"
    status: "Foxwell Digital 자체 데이터 기반. Meta 공식 출처 없음."
    action: "여전히 실무에서 가장 널리 인용되는 기준. 7~10일 분산 옵션 제공."

  - issue: "Andromeda 정확한 culling 임계값 (몇 노출 후 죽이는지)"
    status: "Meta 공식 비공개. 실무 휴리스틱: 최소 3일 + 1,000 노출 후 판단"
    action: "조기 평가 금지 룰(R007)로 보호"

  - issue: "소재당 학습 minimum impressions/conversions"
    status: "Meta 공식 명시 없음. 출처별로 다름 (Madgicx 10전환, AdRow 30~50전환, Meta A/B 100~500전환)"
    action: "단계별 임계값으로 사용 (functional → statistical → high confidence)"

  - issue: "AdRiseLab 티어 권장이 입력 예산에서 strict 한계 초과하는 경우"
    status: "AdRiseLab은 알고리즘 culling 가정. Foxwell은 fair test 가정. 두 가정 다름."
    action: "Foxwell 우선 적용 (보수적), AdRiseLab은 참고용"
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
    - CHANGE_ATTRIBUTION_MODEL
    - CHANGE_PLACEMENT
    - BUDGET_CHANGE_OVER_20PCT
    - CREATE_NEW_CAMPAIGN
    - DELETE_ANYTHING
```

---

## 부록 B: 소재당 학습 진행 상태 (Per-Creative Learning Stages)

소재 1개의 데이터 누적 단계를 5단계로 분류 — 자동 판단 시 사용.

```yaml
per_creative_learning_stages:
  - stage: 0
    name: "출시 직후 (Pre-Live Buffer)"
    condition:
      OR:
        - days_live < 3
        - impressions < 1000
    judgment: "평가 불가"
    action: HOLD
    source: TheOptimizer, AdRow

  - stage: 1
    name: "초기 (Initial Signal)"
    condition:
      AND:
        - days_live >= 3
        - 1000 <= impressions < 5000
        - conversions < 10
    judgment: "방향성만 참고 — 통계 불충분"
    action: MONITOR
    source: AdRow

  - stage: 2
    name: "Functional Minimum (Madgicx 기준)"
    condition:
      AND:
        - conversions >= 10
        - clicks >= 100
    judgment: "기본 판단 가능 — CTR/CVR 트렌드 평가 시작"
    action: APPLY_CREATIVE_RULES_R001_TO_R006
    source: Madgicx

  - stage: 3
    name: "Statistical Significance (AdRow 95%)"
    condition: conversions >= 30
    judgment: "95% 신뢰도 winner 판별 시작 가능"
    action: APPLY_FULL_DECISIONS
    source: AdRow

  - stage: 4
    name: "Foxwell Winner Threshold"
    condition: spend >= (target_cpa * 20)
    judgment: "winner 확정 / 폐기 판별"
    action: FINAL_VERDICT
    source: Foxwell Digital

  - stage: 5
    name: "High Confidence (Meta A/B Tool)"
    condition: conversions >= 100
    judgment: "Meta 공식 A/B 도구 95% 신뢰 도달"
    action: SCALE_OR_KILL_WITH_CONFIDENCE
    source: Coinis / Meta A/B Test
```

---

## 부록 C: 광고세트 셋업 체크리스트 (Live 전 검증)

```yaml
pre_launch_checklist:
  - item: "캠페인 목표"
    value: PURCHASE (전환)
  - item: "성과 목표"
    value: MAXIMIZE_CONVERSION_VALUE  # 전환값 극대화
    requirement: "Purchase value/currency 파라미터 정확 전송"
  - item: "기여 모델"
    value: STANDARD  # 일반
    window: "7일 클릭 + 1일 조회"
  - item: "위치"
    value: 대한민국
  - item: "최소 연령"
    value: 18
  - item: "연령 (추천)"
    value: 18-65+
  - item: "성별"
    value: ALL  # 명확히 단일 성별 제품도 가능하면 ALL
  - item: "상세 타게팅"
    value: NONE  # 비워두기
  - item: "노출 위치"
    value: AUTOMATIC (Advantage+)
  - item: "플랫폼"
    value: 모두 활성화 (FB·IG·Audience Network·Messenger·Threads)
  - item: "인콘텐츠 광고"
    value: EXPANDED
  - item: "Audience Network"
    value: EXPANDED
  - item: "브랜드 안전성"
    value: DEFAULT
  - item: "인벤토리 필터"
    value: DEFAULT
  - item: "퍼블리셔 차단 리스트"
    value: NONE
  - item: "CAPI"
    value: ENABLED + 픽셀 dedup 확인
  - item: "EMQ"
    value: ">= 9.0"
  - item: "마이크로 전환 전송"
    value: ENABLED (PageView, AddToCart, InitiateCheckout, Purchase)
```

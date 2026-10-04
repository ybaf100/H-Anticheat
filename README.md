# H-Anticheat

Geometry Dash의 Megahack, QOL Mod, Eclipse 총 3개의 모드메뉴에서 AREDL 룰셋 기준 치트가 활성화되어있는지를 확인할 수 있는 API입니다.

## API 호출법

HAC는 다른 Geode 모드에서 감지 결과를 읽을 때 사용하는 API입니다. 의존성 ID는 `hwanhee1.hac`이며 공개 헤더는 `HAC.hpp`입니다.

```cpp
#include <hwanhee1.hac/include/HAC.hpp>
```

모든 함수는 게임 메인 스레드에서 호출합니다. 함수 결과는 `geode::Result<T>`입니다. `isErr()`가 참이면 `unwrap()`을 호출하지 말고 오류로 처리하세요.

### 제공 함수

| 함수 | 반환값 | 반환 조건 |
| --- | --- | --- |
| `hac::api::getCurrentState()` | `geode::Result<hac::State>` | 전체 상태를 반환. 요청을 처리할 HAC 리스너가 없거나 API 버전이 맞지 않으면 오류 |
| `hac::api::getSnapshot()` | `geode::Result<hac::Snapshot>` | 공급자·기능별 상세 결과를 반환. 요청을 처리할 HAC 리스너가 없거나 API 버전이 맞지 않으면 오류 |
| `hac::api::hasProhibitedFeaturesEnabled()` | `geode::Result<bool>` | `Active`면 `true`, `Inactive`면 `false`, `Unknown`이면 오류. API 오류도 전달 |
| `hac::api::hasCheatsEnabled()` | `geode::Result<bool>` | `hasProhibitedFeaturesEnabled()`의 별칭 |

### 전체 상태 값과 조건

`getCurrentState()`와 `snapshot.state`는 전체 판정이고, `provider.state`는 공급자 하나의 판정입니다.

| 값 | 개별 `provider.state` | 전체 `snapshot.state` / `getCurrentState()` |
| --- | --- | --- |
| `hac::State::Active` | 해당 공급자에서 현재 활성 금지 기능이 확인됨 | 하나 이상의 공급자에서 금지 기능이 확인됨 |
| `hac::State::Inactive` | 해당 공급자의 검사 항목을 확인했고 활성 금지 기능이나 미확인 기능이 없음. 현재는 주로 QOLMod의 완전한 기본 설정 검사에서 가능 | 내장 집계에서는 공급자 목록이 비었거나 모든 공급자가 `NotPresent`일 때 |
| `hac::State::Unknown` | 버전·기능·조건·조회 범위가 불완전하거나 검증되지 않음 | 활성 금지 기능은 없지만, `NotPresent`가 아닌 공급자가 하나 이상 있음 |
| `hac::State::NotPresent` | 해당 Geode 공급자 모드가 로드되지 않음 | 내장 집계의 전체 상태로는 사용하지 않음 |

전체 상태는 `Active`를 우선합니다. 알려진 금지 기능이 없을 때는 로드된 공급자가 하나라도 있으면 `Unknown`이며, 전부 없을 때만 `Inactive`입니다. 그래서 QOLMod의 개별 `provider.state`가 `Inactive`여도 전체 `snapshot.state`는 `Unknown`일 수 있습니다. `Inactive`는 안전 인증을 뜻하지 않습니다.

오류와 `Unknown`은 다릅니다. `Unknown`은 API가 정상 응답했지만 판정이 불완전하다는 값입니다. API 리스너가 없거나 프로토콜 버전이 맞지 않으면 `Result` 오류가 반환됩니다.

### 현재 상태 확인 예제

```cpp
void checkHacState() {
    auto result = hac::api::getCurrentState();
    if (result.isErr()) {
        // HAC API를 사용할 수 없거나 API 버전이 맞지 않음
        return;
    }

    switch (result.unwrap()) {
        case hac::State::Active:
            // 확인된 금지 기능이 있음
            break;
        case hac::State::Unknown:
            // 감지 대상은 있지만 전체 판정을 확정할 수 없음
            break;
        case hac::State::Inactive:
            // 현재 감지 대상 공급자가 없음
            break;
        case hac::State::NotPresent:
            // 전체 집계에서는 사용하지 않으며, 보통 공급자별 상태에 나타남
            break;
    }
}
```

### 상세 스냅샷 값

`getSnapshot()`은 아래 `Snapshot`을 반환합니다. 문자열과 배열은 해당 스냅샷이 소유합니다.

| 값 | 의미와 조건 |
| --- | --- |
| `apiVersion` | 공개 API 프로토콜 버전. 현재 `2` |
| `revision` | 새 감지 표본을 만들 때마다 증가하는 번호 |
| `sampledAtMonotonicMs` | 표본 생성 시각(단조 증가 시계의 밀리초, 실제 날짜/시각 아님) |
| `state` | 위 표의 전체 상태 |
| `anyHackEnabled` | 현재 활성 금지 기능이 하나 이상 있으면 `true` |
| `hasUnknownProviders` | `NotPresent`가 아닌 공급자가 하나라도 있으면 `true`. 공급자별 상태가 `Inactive`여도 전체 판정이 완전하다는 뜻은 아님 |
| `providerReportedAttemptCheat` | 공급자가 시도 이력을 지원하고, 공급자 자체가 이번 시도에서 치트를 보고했으면 `true` |
| `providers` | Eclipse, QOLMod, Mega Hack의 공급자별 결과 |
| `policyId`, `policySource` | 적용한 판정 정책 ID와 기준 링크 |
| `mode` | 현재 플레이 모드. `Classic` 또는 `Platformer`; 플레이 중이 아닐 때 기본값은 `Classic` |
| `prohibitedInAttempt` | HAC가 이번 플레이 시도에서 금지 기능을 확인한 이력이 있으면 `true`. 기능을 꺼도 해당 시도 동안 유지될 수 있음 |
| `attemptFeatures` | 이번 시도에서 확인된 금지 기능 이력 목록 |

각 `ProviderSnapshot`에는 `id`, `name`, `version`, `state`, `coverage`, `detail`과 다음 목록·상태가 있습니다.

| 공급자 값 | 의미 |
| --- | --- |
| `enabledFeatures` | 현재 활성화된 금지 기능만 포함 |
| `features` | 허용·금지·조건부·미분류·미확인 등 전체 관찰 항목 |
| `attemptFeatures` | 공급자가 보고한 시도 관련 항목 |
| `attemptHistoryAvailable` | 공급자의 시도 이력을 조회할 수 있으면 `true` |
| `cheatedInAttempt` | 공급자 자체가 보고한 시도 중 치트 여부 |
| `providerReportedEnabled` | 공급자 자체 집계에서 활성 상태를 보고했는지 여부. HAC 판정과 같지는 않음 |
| `hasUnknownFeatures` | 기능 목록에 `Unknown` 항목이 하나 이상 있으면 `true` |
| `catalogSize` | 해당 공급자에서 검사 대상으로 사용하는 카탈로그 항목 수 |

`coverage`는 조회 근거를 나타냅니다: `ProviderAPI`는 공급자 공식 API, `KnownFeatures`는 알려진 기능 목록의 부분 조회, `FullBuiltinConfiguration`은 QOLMod 기본 설정 전체 조회, `Unsupported`는 미지원 또는 검증되지 않은 조회입니다.

각 `Feature`는 `id`, `name`, `state`, `policy`, `evidence`, `value`, `source`, `detail`을 가집니다. `state`는 항목의 활성·비활성·미확인 상태이고, `policy`는 `Allowed`, `Prohibited`, `Conditional`, `Unmapped`, `NotApplicable` 중 적용된 분류입니다. `evidence`(`EvidenceKind`)는 `RuntimeAPI`(실행 중 API 조회) 또는 `Configuration`(저장 설정 조회)이며, `value`, `source`, `detail`은 읽은 값·판정 기준·추가 설명입니다.

`Mode`는 `Classic` 또는 `Platformer`입니다. `Coverage`와 `Feature` 분류는 항목마다 확인 가능한 범위가 달라질 수 있으므로, 설명이 필요하면 `detail`과 `source`도 함께 표시하세요.

### 상세 기능 조회 예제

```cpp
void showHacDetails() {
    auto result = hac::api::getSnapshot();
    if (result.isErr()) return;

    auto const& snapshot = result.unwrap();
    if (snapshot.prohibitedInAttempt) {
        // 이번 시도 중 과거에 확인된 금지 기능이 있음
    }

    for (auto const& provider : snapshot.providers) {
        for (auto const& feature : provider.enabledFeatures) {
            // 현재 금지 기능: provider.name, feature.name, feature.detail
        }
    }
}
```

### 금지 기능 여부만 확인하는 예제

```cpp
void checkForProhibitedFeatures() {
    auto result = hac::api::hasProhibitedFeaturesEnabled();
    if (result.isErr()) return; // Unknown 또는 API 오류

    bool prohibited = result.unwrap();
    // true: 확인된 금지 기능 있음, false: 이번 표본에서 확인된 금지 기능 없음
}
```

HAC 감지 표본은 기본적으로 약 100ms마다 갱신됩니다. API 함수 호출은 즉시 새 표본을 요청하므로, 게임 메인 스레드에서 필요한 시점에 호출하세요.






Hwanhee1-Anticheat

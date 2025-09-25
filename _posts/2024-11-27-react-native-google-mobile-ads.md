---
title: react-native-google-mobile-ads 구현하기 🎯
date: 2024-11-27 16:00:00 +0900
categories: [개발일지, React Native]
tags: [expo, react-native, google-mobile-ads, admob, rewarded-ad, 광고]
pin: false
---

## 🚀 시작하며

앱에 수익 모델을 도입하기 위해 기존 Expo RN 프로젝트에 Google Mobile Ads를 적용하게 되었다.

**리워드형 전면 광고(Rewarded Interstitial Ad)**를 구현하는 과정에서 마주친 여러 이슈들과 해결 방법을 기록해둔다.

---

## 1️⃣ 기본 설정 및 라이브러리 설치

가장 기본적인 설치부터 시작했다.

```bash
yarn add react-native-google-mobile-ads
```

### Expo build-properties 설정

```json
// app.config.js
{
  "expo": {
    "plugins": [
      [
        "expo-build-properties",
        {
          "ios": {
            "useFrameworks": "static"
          }
        }
      ]
    ]
  }
}
```

---

## 2️⃣ 광고 상태에 따른 로직 처리를 위한 상태값 핸들링

광고 상태에 따른 로직 처리를 위해 상태값을 핸들링했다.

### 🔧 해결 방법
필수적인 상태값을 명확히 정의했다:

```javascript
/**
 * 광고 상태 정보
 * @typedef {Object} AdStateInfo
 * @property {boolean} isAdLoading - 광고 로딩 중
 * @property {boolean} isRewarded - 리워드 획득
 * @property {boolean} isEndVideo - 광고 종료
 * @property {boolean} isEmptyInventory - 재고 부족
 */
const [adStateInfo, setAdStateInfo] = useStoreState(adStateInfoState)
```

---

## 3️⃣ 플랫폼별 Ad Unit ID 관리 이슈

iOS와 Android의 Ad Unit ID가 다른데 이를 플랫폼 별 분기 설정을 했다.

```javascript
// After ✅ - 플랫폼 분기
const currentAdUnitId = Platform.OS === 'android'
  ? 'ca-app-pub-xxxxxxxxxxxxxxxx/android-id'
  : 'ca-app-pub-xxxxxxxxxxxxxxxx/ios-id'
```

---

## 4️⃣ 광고 이벤트 리스너 관리 지옥

광고에는 수많은 이벤트가 있는데, 각각을 적절히 처리하지 않으면 예상치 못한 문제가 발생한다.
ERROR_CODE_INTERNAL_ERROR: 내부 서버 오류나 예상치 못한 문제가 발생했을 때 표시됩니다.
ERROR_CODE_INVALID_REQUEST: 잘못된 광고 요청입니다. 잘못된 광고 단위 ID 사용, 요청에 필요한 설정 누락 등으로 인해 발생할 수 있습니다.
ERROR_CODE_NETWORK_ERROR: 네트워크 연결 문제나 요청 중 타임아웃이 발생한 경우입니다.
ERROR_CODE_NO_FILL: 광고 인벤토리가 없어서 요청을 충족할 수 없을 때 발생합니다.
ERROR_CODE_AD_REUSED: 이미 사용된 광고를 다시 사용하려고 할 때 발생합니다. 동일한 광고 인스턴스를 두 번 이상 사용하려 할 때 자주 나타납니다.
ERROR_CODE_MEDIATION_NO_FILL: 미디에이션에 포함된 광고 네트워크에서 유효한 광고를 제공하지 못했을 때 발생합니다.
ERROR_CODE_MEDIATION_ADAPTER_ERROR: 특정 미디에이션 어댑터가 광고 로드에 실패했을 때 발생합니다. 추가 정보는 userInfo에 포함될 수 있습니다.
ERROR_CODE_APP_ID_MISSING: AdMob 설정에서 앱 ID가 누락되었거나 유효하지 않을 때 발생합니다.
ERROR_CODE_AD_LIMIT_REACHED: 사용자가 일정 시간 동안 보여질 수 있는 광고의 제한을 초과했을 때 발생합니다.


### 🔧 해결 방법
체계적인 이벤트 리스너 구조를 만들었다:

```javascript
const loadAdmob = useCallback(async () => {
  setIsAdLoading(true)

  try {
    const admob = RewardedInterstitialAd.createForAdRequest(currentAdUnitId, {
      requestNonPersonalizedAdsOnly: true, // GDPR 준수
    })

    // 1) 리워드 획득 감지
    admob.addAdEventListener(RewardedAdEventType.EARNED_REWARD, reward => {
      if (reward?.amount > 0) {
        setIsRewarded(true)
      }
    })

    // 2) 광고 로드 완료 → 즉시 실행
    admob.addAdEventListener(RewardedAdEventType.LOADED, () => {
      StatusBar?.setHidden(true) // 디바이스 상태바 숨김
      admob?.show()
    })

    // 3) 광고 종료 처리
    admob.addAdEventListener(AdEventType.CLOSED, () => {
      StatusBar?.setHidden(false)
      setIsEndVideo(true)
      admob.removeAllListeners() // 메모리 누수 방지
    })

    // 4) 에러 처리
    admob.addAdEventListener(AdEventType.ERROR, error => {
      StatusBar?.setHidden(false)
      handleAdError(error)
      setIsAdLoading(false)
    })

    admob?.load()
  } catch (error) {
    console.error('AdMob 로드 에러:', error)
    setIsAdLoading(false)
  }
}, [])
```

---

## 5️⃣ 광고 재고 부족 (No Fill) 지옥

특정 시간대나 지역에서 `no-fill` 에러가 계속 발생해서 사용자가 광고를 볼 수 없는 상황이 발생했다.

### 🔧 해결 방법
하루 단위로 재고 부족 상태를 기록하고 사용자에게 친화적인 안내를 제공했다:

```javascript
const handleAdError = error => {
  const { code = '', message } = error || {}

  if (code?.includes('no-fill')) {
    // 하루 단위로 재고 부족 상태 저장
    AsyncStorage.setItem(
      'EMPTY_INVENTORY_DATE', 
      JSON.stringify(dayjs().format('YYYY-MM-DD'))
    )
    setIsEmptyInventory(true)
    showToast('오늘은 준비한 광고가 모두 소진되었어요.\n내일 또 오세요!')
  } else {
    showToast('광고를 불러오는데 실패했어요. 잠시 후 다시 시도해주세요.')
  }
}

// 앱 시작 시 재고 상태 확인
const checkEmptyInventory = async () => {
  const cancelDate = JSON.parse(await AsyncStorage.getItem('EMPTY_INVENTORY_DATE'))
  const isNextDay = !cancelDate ? true : dayjs().diff(cancelDate, 'day') >= 1

  if (!isNextDay) {
    setIsEmptyInventory(true)
  }
}
```

---

## 6️⃣ StatusBar 깜빡임 이슈

광고 재생 중 StatusBar가 보였다 안 보였다 하면서 사용자 경험을 해쳤다.

### 🔧 해결 방법
광고 생명주기에 맞춰 StatusBar를 정확히 제어했다:

```javascript
// 광고 시작 시
admob.addAdEventListener(RewardedAdEventType.LOADED, () => {
  StatusBar?.setHidden(true) // 전체화면 모드
  admob?.show()
})

// 광고 종료 시
admob.addAdEventListener(AdEventType.CLOSED, () => {
  StatusBar?.setHidden(false) // 상태바 복원
  setIsEndVideo(true)
})

// 에러 발생 시에도 반드시 복원
admob.addAdEventListener(AdEventType.ERROR, error => {
  StatusBar?.setHidden(false) // 에러 시에도 복원 보장
  handleAdError(error)
})
```

---

## 7️⃣ 리워드 중복 지급 방지

광고를 끝까지 보지 않았는데도 리워드가 지급되거나, 중복으로 지급되는 문제가 발생했다.

### 🔧 해결 방법
리워드 획득과 광고 종료가 모두 완료되었을 때만 처리하도록 했다:

```javascript
// 두 조건이 모두 만족되어야만 리워드 지급
useEffect(() => {
  if (isRewarded && isEndVideo) {
    processReward()
  }
}, [isRewarded, isEndVideo])

const processReward = async () => {
  try {
    const response = await rewardAPI.giveReward({
      type: 'ad_reward',
      amount: 100,
    })

    if (response.success) {
      showRewardToast('리워드를 받았어요! 🎉')
    }
  } catch (error) {
    console.error('리워드 처리 에러:', error)
  } finally {
    // 상태 초기화로 중복 방지
    setIsAdLoading(false)
    setIsRewarded(false)
    setIsEndVideo(false)
  }
}
```

---

## 8️⃣ 메모리 누수와 중복 호출 방지

광고 리스너들이 제대로 정리되지 않아서 메모리 누수가 발생하고, 사용자가 버튼을 연타하면 중복 호출되는 문제가 있었다.

### 🔧 해결 방법

#### 메모리 누수 방지:
```javascript
admob.addAdEventListener(AdEventType.CLOSED, () => {
  StatusBar?.setHidden(false)
  setIsEndVideo(true)
  admob.removeAllListeners() // 모든 리스너 정리
})

useEffect(() => {
  return () => {
    clearTimeout(refetchTimer.current) // 타이머 정리
  }
}, [])
```

#### 중복 호출 방지:
```javascript
const loadAdmob = useCallback(async () => {
  if (isAdLoading) return // 이미 로딩 중이면 중단

  setIsAdLoading(true)
  // ... 광고 로드 로직
}, [isAdLoading])

// UI에서도 비활성화
<Button
  title={isAdLoading ? '광고 준비 중...' : '광고 보고 리워드 받기'}
  disabled={isAdLoading || isEmptyInventory} // 로딩 중 비활성화
  onPress={loadAdmob}
/>
```

---

## 🎯 마치며

무엇보다 Expo 클라우드 빌드를 사용하고있는 조건때문에 각 OS별 빌드 환경을 컨트롤해야하는 까다로움이 있을 줄 알았는데 다행히도 간단한 Expo 빌드옵션 추가로 해결할 수 있었다.

### 🎉 좋아진 점들
- **수익 모델 확보**: 안정적인 광고 수익 창출 가능

### 💡 교훈
- **테스트 환경 활용**: 개발 중에는 반드시 테스트 Ad Unit ID 사용 (안그러면 막힘)
- **에러 로깅 시스템**: 프로덕션에서 광고 관련 이슈 추적 필수 (상황별 유저 대응을 위해)

---

> **관련 링크**  
> - [React Native Google Mobile Ads 공식 문서](https://docs.page/invertase/react-native-google-mobile-ads)
> - [AdMob 정책 가이드](https://support.google.com/admob/answer/6128543)

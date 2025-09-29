---
title: react-native-google-mobile-ads 구현하기
date: 2024-11-27 16:00:00 +0900
categories: [개발일지, React Native]
tags: [expo, react-native, google-mobile-ads, admob, rewarded-ad, 광고]
---

농장 게임 프로젝트도 어느정도 안정화 되었고, 유저들에게 더 큰 혜택을 제공해주기 위해서는 우리도 수익화 모델을 구상해야했다.
유저들이 점점 많이 늘어남에 따라 우리가 제공해줄 수 있는 리소스들은 한정적이였고 가장 먼저 쉽게 넣을 수 있는 것은 구글애드몹 이였다.

하지만 우리의 프로젝트는 Expo React-Native 프로젝트였고, 그에 맞는 라이브러리를 찾아야 했고 그게 바로 react-native-google-mobile-ads 였다!
생각보다 적용 방법이 쉬워서 문제는 없었지만
적용하고나서 광고 호출 상태에 따른 대응이 필요했어서 호출 결과에 따라 유저에게 대응하는 로직을 추가해주어야 했다.

---

## 기본 설정 및 라이브러리 설치

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

## 플랫폼별 Ad Unit ID 관리

한개의 앱에 여러 형태의 광고를 넣어야되기 때문에 각 서비스 별 구현로직 상단에 unit-id를 별도로 선언 해야한다.

```javascript
// After ✅ - 플랫폼 분기
const currentAdUnitId = Platform.OS === 'android'
  ? 'ca-app-pub-xxxxxxxxxxxxxxxx/android-id'
  : 'ca-app-pub-xxxxxxxxxxxxxxxx/ios-id'
```

---

## 광고 상태에 따른 로직 처리를 위한 상태값 핸들링

광고 상태에 따른 로직 처리를 위해 상태값을 핸들링했다.
광고에는 수많은 에러 이벤트가 있는데, 여기에서 보통 `no-fill` 키워드가 대부분이다.

ERROR_CODE_INTERNAL_ERROR: 내부 서버 오류나 예상치 못한 문제가 발생했을 때 표시됩니다.
ERROR_CODE_INVALID_REQUEST: 잘못된 광고 요청입니다. 잘못된 광고 단위 ID 사용, 요청에 필요한 설정 누락 등으로 인해 발생할 수 있습니다.
ERROR_CODE_NETWORK_ERROR: 네트워크 연결 문제나 요청 중 타임아웃이 발생한 경우입니다.
ERROR_CODE_NO_FILL: 광고 인벤토리가 없어서 요청을 충족할 수 없을 때 발생합니다.
ERROR_CODE_AD_REUSED: 이미 사용된 광고를 다시 사용하려고 할 때 발생합니다. 동일한 광고 인스턴스를 두 번 이상 사용하려 할 때 자주 나타납니다.
ERROR_CODE_MEDIATION_NO_FILL: 미디에이션에 포함된 광고 네트워크에서 유효한 광고를 제공하지 못했을 때 발생합니다.
ERROR_CODE_MEDIATION_ADAPTER_ERROR: 특정 미디에이션 어댑터가 광고 로드에 실패했을 때 발생합니다. 추가 정보는 userInfo에 포함될 수 있습니다.
ERROR_CODE_APP_ID_MISSING: AdMob 설정에서 앱 ID가 누락되었거나 유효하지 않을 때 발생합니다.
ERROR_CODE_AD_LIMIT_REACHED: 사용자가 일정 시간 동안 보여질 수 있는 광고의 제한을 초과했을 때 발생합니다.

---

### 구현 방법

이벤트 리스너 구조를 만든다.

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

## 광고 재고 부족 (No Fill)

특정 시간대나 지역에서 `no-fill` 에러가 계속 발생해서 사용자가 광고를 볼 수 없는 상황이 발생했다.
그래서 하루 단위로 재고 부족 상태를 기록하고,
사용자에게 지속적인 에러 가능성을 열어주는 것보다는 다음날 까지는 참여 할 수 없도록 유도하고
다음날 다시 참여할 수 있도록 호출을 방지했다.

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

## StatusBar가 광고 닫기 버튼과 겹쳐지는 이슈

광고 재생 중 Edge-to-Edge 디바이스 환경인 경우 StatusBar가 광고의 닫기 버튼 영역과 겹쳐지는 이슈로 광고가 닫히지 않는 현상이 제보되었다.
단순히 광고 재생을 시키기전에 hidden 처리를 하고, 광고가 종료되면 hidden 처리를 해제하면 된다.

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

## 마무리

무엇보다 Expo 클라우드 빌드를 사용하고있는 조건때문에 각 OS별 빌드 환경을 컨트롤해야하는 까다로움이 있을 줄 알았는데 다행히도 간단한 Expo 빌드옵션 추가로 수월하게 설치할 수 있었다.
이로써 작지만 좋은 수익화 모델을 경험할 수 있었습니다~

---

**관련 링크**  
> - [React Native Google Mobile Ads 공식 문서](https://docs.page/invertase/react-native-google-mobile-ads)
> - [AdMob 정책 가이드](https://support.google.com/admob/answer/6128543)

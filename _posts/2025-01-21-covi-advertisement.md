---
title: WebView 기반 광고 SDK 연동하기
date: 2025-01-21 16:00:00 +0900
categories: [개발일지, React Native]
tags: [react-native, webview, 광고sdk, postmessage, 하이브리드앱]
---

외부 광고 업체에서 우리 게임 서비스에 광고를 연동해달라고 연락이 왔다!
아무래도 유저의 수가 엄청나게 많아지니 이런 좋은 수익화 모델들이 찾아오는것 같다!
하지만 이 업체의 SDK는 웹 전용 밖에 제공하지않는다.

결국 React Native 프로젝트에서 **웹 SDK**를 통합해야 하는 상황이 생겼다.
그래서 **웹 SDK**를 WebView와 postMessage를 활용해 구현을 하기로 했다.
WebView 기반 하이브리드 방식으로 구현하는 과정을 기록해두려 한다!

---

## 기본 WebView 구현

가장 기본이 되는 WebView 컴포넌트 구성부터 시작했다.

```javascript
import WebView from 'react-native-webview'

const AdWebView = ({ url, onMessage }) => {
	const webViewRef = useRef(null)

	return (
		<WebView
			ref={webViewRef}
			originWhitelist={['*']}
			source={{ uri: url }}
			onLoadEnd={sendMessage}
			onMessage={handleMessage}
			onShouldStartLoadWithRequest={handleNavigation}
			mixedContentMode="always"
			allowsInlineMediaPlayback={true}
			scrollEnabled={false}
		/>
	)
}
```

---

## 환경별 URL 설정

광고를 실질적으로 재생시키는 웹페이지에서
처음에 SDK를 초기화할 때 개발용 APP-ID와 프로덕션용 APP-ID를 적용하기 위해
현재 웹뷰가 요청하고있는 환경이 어떤 환경에서 요청하고있는지 서브도메인으로 분기 처리를 해놓았다.

그래서 웹뷰에서는 그 환경에 맞는 URL을 동적으로 생성해주었다.

```javascript
const webViewUrl = useMemo(() => {
	const baseUrl = isProduction
		? 'https://prod-api.example.com'
		: 'https://stage-api.example.com'

	const platform = Platform.OS === 'android' ? 'aos' : 'ios'

	return `${baseUrl}/wv/rn/ad-player?target=myapp_${platform}`
}, [])
```

---

## postMessage 데이터 전송

현재 유저의 정보를 SDK 초기화 데이터에 같이 전달해주면 최적화된 광고를 보여줄 수 있다고 한다. (물론 선택 사항이였지만)
그래서 RN WebView에서 웹페이지에 사용자 정보를 전달해야하는데, postMessage를 활용해 전달했다.

```javascript
const sendUserData = useCallback(() => {
  const userData = {
    target: Platform.OS === 'android' ? 'myapp_aos' : 'myapp_ios',
    adid: advertisingId,
    gender: userGender?.slice(0, 1)?.toUpperCase(),
    age: calculateAge(userBirth),
  }

  const message = JSON.stringify(userData)
  webViewRef.current?.postMessage(message)
}, [advertisingId, userGender, userBirth])

// WebView 로드 완료 시 데이터 전송
<WebView
  onLoadEnd={sendUserData}
  // ... 기타 props
/>
```

---

## WebView 메시지 처리

이제 광고를 재생하는 웹페이지에서 광고를 초기화하고 재생을 하는 과정에서 발생하는 여러 상태들을 문자열로 전달받아서 처리해주어야 한다.
예를들어 광고가 재생되고있는지 광고를 불러올수없는지... 등등에 대한 상태를 말한다.
이것 또한 메시지로 받아서 파싱해서 처리했다!

```javascript
const handleMessage = useCallback(event => {
	const message = JSON.parse(event?.nativeEvent?.data)
	const { type, data } = message || {}

	switch (type) {
		case 'PLAYER_VIDEO_PLAY':
			// 광고 재생 시작
			break

		case 'PLAYER_VIDEO_ENDED':
			// 광고 재생 완료
			break

		case 'PLAYER_NO_ADS':
			// 광고 없음 - 다른 광고 소스 탐색
			break

		case 'PLAYER_VIDEO_PAUSE':
			// 광고 일시정지
			break
	}
}, [])
```

---

## 외부 링크 처리

웹뷰 컴포넌트에서 광고를 클릭해서 해당 광고링크로 이동했을때
예상치 못한 URL 또는 스크립트가 동작해서 광고주가 유도한 유저 플로우대로 이동하지 않을 것을 예방해서
별도의 의미없는 외부 링크들을 무시하도록 동작을 추가했다.

```javascript
const handleNavigation = useCallback(request => {
	const { url } = request || {}

	// 광고 플레이어 URL은 허용
	if (url.includes('wv/rn/ad-player')) {
		return true
	}

	// about:blank 차단
	if (url.startsWith('about:')) {
		return false
	}

	// 앱 스키마 처리
	if (!url.startsWith('http')) {
		Linking.canOpenURL(url).then(supported => {
			if (supported) {
				Linking.openURL(url)
			}
		})
		return false
	}

	// 외부 웹페이지는 브라우저에서 열기
	if (url.startsWith('http')) {
		Linking.openURL(url)
		return false
	}

	return true
}, [])
```

---

## 세로형 광고 전체화면 구현

지금까지는 가로형 작은 영상 광고였지만, 세로형 전면 광고 일 경우에는 스와이프를 통해 광고를 닫을 수 있도록 구현해야했다.
당연히 웹뷰는 현재 인앱의 동작과 별도의 동작으로 인식하기 때문에, 현재 최상위 제스처가 좌우 스와이프가 동작했을때 
웹뷰동작이 아닌 웹뷰 컴포넌트가 좌우로 사라지는 동작으로 인식하도록 구현했다.

#### 웹뷰 컴포넌트에 나타나고 사라지는 reanimated 애니메이션 적용

```javascript
const VerticalAdModal = ({ isVisible, onClose }) => {
	const { height: windowHeight, width: windowWidth } = useWindowDimensions()
	const insets = useSafeAreaInsets()

	return (
		<Animated.View className="absolute top-0 w-full h-full bg-black" style={[animatedStyle]}>
			<View style={{ height: windowHeight - insets.top - insets.bottom }}>
				<WebView
					source={{ uri: verticalAdUrl }}
					// ... 기타 설정
				/>
				<CloseButton onPress={onClose} />
			</View>
		</Animated.View>
	)
}
```

#### 스와이프 제스처 닫기

```javascript
import { Gesture, GestureDetector } from 'react-native-gesture-handler'
import Animated, { useSharedValue, useAnimatedStyle, withSpring } from 'react-native-reanimated'

const SwipeToCloseAd = ({ children, onClose }) => {
	const translateY = useSharedValue(0)

	const swipeGesture = Gesture.Pan()
		.onUpdate(event => {
			// 위로만 스와이프 허용
			translateY.value = Math.min(0, event.translationY)
		})
		.onEnd(() => {
			if (translateY.value < -150) {
				// 충분히 스와이프했으면 닫기
				translateY.value = withSpring(-windowHeight, {}, () => {
					runOnJS(onClose)()
				})
			} else {
				// 원위치
				translateY.value = withSpring(0)
			}
		})

	const animatedStyle = useAnimatedStyle(() => ({
		transform: [{ translateY: translateY.value }],
	}))

	// children으로 웹뷰 컴포넌트를 전달
	return (
		<GestureDetector gesture={swipeGesture}>
			{children}
		</GestureDetector>
	)
}
```

---

## 마무리

이렇게 부분적인 하이브리드 형태의 구현은 처음이지만 신선했다!
그리고 postMessage 타이밍과 데이터가 잘 전달되는지 확인하는 과정에서 시간을 많이 잡아먹긴했다.
하지만 WebView 기반 광고 SDK 통합을 통해 광고 수익을 창출할 수 있었다!
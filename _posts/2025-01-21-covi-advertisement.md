---
title: WebView 기반 광고 SDK 연동하기
date: 2025-01-21 16:00:00 +0900
categories: [개발일지, React Native]
tags: [react-native, webview, 광고sdk, postmessage, 하이브리드앱]
pin: false
---

## 🚀 시작하며

React Native 프로젝트에서 **웹 전용 광고 SDK**를 통합해야 하는 상황이 생겼다. 광고 업체에서 React Native용 SDK를 제공하지 않고 **웹 SDK만 제공**하는 상황에서, WebView와 postMessage를 활용해 문제를 해결한 경험을 공유한다.

WebView 기반 하이브리드 방식으로 구현하는 과정에서 마주친 8가지 주요 이슈들과 해결 방법을 기록해둔다.

---

## 1️⃣ 기본 WebView 설정

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

## 2️⃣ 환경별 URL 설정 이슈

개발/프로덕션 환경에 따른 동적 URL 생성이 필요했다.

### 🔧 해결 방법

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

## 3️⃣ postMessage 데이터 전송 이슈

React Native에서 WebView로 사용자 정보를 전달해야 하는데, 타이밍과 형식을 고려해야한다.

### 🔧 해결 방법

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

## 4️⃣ WebView 메시지 처리 이슈

WebView에서 오는 광고 상태 메시지를 적절히 처리해야 한다.

### 🔧 해결 방법

```javascript
const handleMessage = useCallback(event => {
	const message = JSON.parse(event?.nativeEvent?.data)
	const { type, data } = message || {}

	switch (type) {
		case 'PLAYER_VIDEO_PLAY':
			// 광고 재생 시작
			trackAdEvent('ad_started')
			setAdPlaying(true)
			break

		case 'PLAYER_VIDEO_ENDED':
			// 광고 재생 완료
			trackAdEvent('ad_completed')
			setAdPlaying(false)
			onAdComplete?.()
			break

		case 'PLAYER_NO_ADS':
			// 광고 없음 - 다른 광고 소스 탐색
			findAlternativeAd()
			break

		case 'PLAYER_VIDEO_PAUSE':
			// 광고 일시정지
			setAdPlaying(false)
			break

		default:
			console.log('Unknown message type:', type)
	}
}, [])
```

---

## 5️⃣ 외부 링크 처리 이슈

광고 클릭 시 예상치 못한 URL로 이동하거나 앱이 크래시되는 문제를 방지하기 위해 외부 링크 처리를 구현했다.

### 🔧 해결 방법

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

## 6️⃣ 세로형 광고 전체화면 구현 이슈

세로형 광고는 전체화면 모달 방식으로 구현해야 했는데, 화면 크기와 Safe Area 처리가 필요.

### 🔧 해결 방법

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

---

## 7️⃣ 스와이프 제스처 닫기 이슈

사용자 친화적인 스와이프 닫기 기능을 구현하는 과정에서 애니메이션과 제스처 충돌이 발생했다.

### 🔧 해결 방법

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

	return (
		<GestureDetector gesture={swipeGesture}>
			<Animated.View style={animatedStyle}>{children}</Animated.View>
		</GestureDetector>
	)
}
```

---

## 8️⃣ 동적 애니메이션과 메모리 누수 예방

광고 시작 시 부드러운 페이드인 효과를 구현하면서 메모리 누수 예방을 위해 여러 초기화 로직을 추가했다.

### 🔧 해결 방법

```javascript
const fadeValue = useSharedValue(0)

const fadeInStyle = useAnimatedStyle(() => ({
	opacity: interpolate(fadeValue.value, [0, 1], [0, 1]),
	transform: [
		{
			translateX: interpolate(fadeValue.value, [0, 1], [windowWidth, 0]),
		},
	],
}))

// 광고 시작 시 애니메이션 실행
const startFadeAnimation = () => {
	fadeValue.value = withTiming(1, { duration: 300 })
}

// 메모리 누수 방지
useEffect(() => {
	return () => {
		// 타이머 정리
		clearTimeout(animationTimer.current)

		// 애니메이션 정리
		cancelAnimation(fadeValue)
		cancelAnimation(translateY)

		// 값 초기화
		fadeValue.value = 0
		translateY.value = 0
	}
}, [])
```

---

## 🎯 마치며

WebView 기반 광고 SDK 통합을 통해 광고 수익을 창출할 수 있었다.

### 😤 가장 힘들었던 부분들
- **postMessage 타이밍 이슈**: WebView 로드 완료와 메시지 전송 타이밍 맞추기
- **크로스 플랫폼 호환성**: iOS/Android 간 WebView 동작 차이

### 🎉 좋아진 점들
- **사용자 경험 향상**: 스와이프 제스처와 애니메이션으로 웹뷰와 네이티브 사이의 일체형 UX 제공
- **광고 수익 창출**: 광고 수익을 창출

### 💡 교훈
- **메시지 전달 포스트 & 리스너**: WebView와 Web사이의 유저정보와 광고응답 포스트 메시지를 연동하는 과정을 견고하게 만들기

---

> **관련 링크**  
> - [React Native WebView 공식 문서](https://github.com/react-native-webview/react-native-webview)
> - [postMessage API 가이드](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage)

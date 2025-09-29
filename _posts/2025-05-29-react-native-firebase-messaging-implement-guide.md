---
title: 'React Native Firebase Messaging (FCM + notifee) 구현 가이드! for Android'
date: 2025-05-29 16:00:00 +0900
category: [개발일지, React Native]
tags: [Expo, FCM, Firebase, 푸시알림, react-native-firebase, 메시징, 개발가이드]
---

원래는 프로젝트 성격상 Expo-notifications 를 사용하면 된다!
하지만 AOS 디바이스에서 동작해야 할 푸시 형태를 다양하게 사용하기위해 FCM과 Notifee 라이브러리를 동시에 채용해버렸습니다!

**Firebase Cloud Messaging(FCM + notifee)** 을 구현해보자!

---

## 설치 및 초기 설정

### 1. 패키지 설치

```bash
# Firebase App 모듈
yarn add @react-native-firebase/app

# Firebase Messaging 모듈
yarn add @react-native-firebase/messaging

# Notifee (Android 알림 고급 기능용)
yarn add @notifee/react-native
```

### 2. Notifee 설치 및 설정

#### Expo 프로젝트에서 Notifee 사용

**app.json 또는 app.config.js에 플러그인 추가**

```json
{
	"name": "my app",
	"plugins": ["@notifee/react-native"]
}
```

**중요:** Notifee는 Java JDK 11+가 필요합니다. 개발 빌드 시 Eas cloud build 환경이 다르기 때문에 eas.json에 따로 설정해줘야 합니다!

```json
{
	"build": {
		"dev": {
			"developmentClient": true,
			"distribution": "internal",
			"android": {
				"image": "ubuntu-18.04-jdk-11-ndk-r19c"
			}
		}
	}
}
```

### 3. 기본 Firebase 앱 인스턴스 생성

```javascript
// notificationUtil.js

import { getApp } from '@react-native-firebase/app'
import { getMessaging } from '@react-native-firebase/messaging'

/** Firebase App 인스턴스 */
const firebaseApp = getApp()

/** Firebase Messaging 인스턴스 */
const messaging = getMessaging(firebaseApp)

/** notifee 데이터 (Android 전용으로 notifee import 자체가 분기처리 되어있어서 let으로 선언) */
let notifeeData = {}
```

---

## 핵심 구현 사항들

### 1. 메시지 토큰 관리

#### 토큰 생성 및 갱신

```javascript
import { deleteToken, getToken } from '@react-native-firebase/messaging'

/**
 * 기존 푸시코드를 삭제하고 새로운 푸시코드를 가져온다.
 *
 * @returns {Promise<string>} Push Token
 */
export async function getPushTokenAsync() {
	try {
		// 개발모드에서 사용 금지
		if (!Device.isDevice) {
			return '' // 개발모드에서 사용 금지
		}

		// Firebase 메시징 토큰 삭제 후 다시 가져오기
		await deleteToken(messaging)
		const token = await getToken(messaging)

		return token || '' // Firebase에서 받은 토큰을 반환, 없을 경우 빈 문자열 반환
	} catch (error) {
		console.error('Error fetching push token:', error)
		return '' // 오류 발생 시 빈 문자열 반환
	}
}
```

### 2. Foreground 메시지 처리

#### 앱이 활성 상태일 때 메시지 수신

```javascript
// notificationUtil.js

import { onMessage } from '@react-native-firebase/messaging'

let notifeeData = {}

/**
 * Notifee 라이브러리를 초기화한다.
 *
 * Android 일 때에만 사용함
 */
export async function initNotifeeData() {
	return await import('@notifee/react-native')
}

/** Notifee Event 에 대한 동작을 설정한다. */
const setNotifeeEventAction = async ({ type, detail }) => {
	const { default: notifee, EventType } = notifeeData
	const { notification, pressAction } = detail || {}
	switch (type) {
		case EventType.DELIVERED:
			// 푸시 수신
			break
		case EventType.PRESS:
			// 푸시 클릭 (notification 내부 데이터를 반드시 확인해보고 동작 시키기!)
			setPushClickEvent(notification)
			await notifee.cancelNotification(notification.id)
			break
	}
}

/** Notifee Background Event 설정 */
export async function setNotifeeBackgroundEvent() {
	notifeeData = await initNotifeeData()
	const { default: notifee } = notifeeData
	notifee.onBackgroundEvent(setNotifeeEventAction)
}

/** Notification 데이터를 알람으로 띄워주기 */
async function onMessageReceived(remoteMessage) {
	try {
		const {
			push_type: pushType,
			view_type: viewType,
			image_uri: imageUri,
			landing_uri: landingUri,
			expanded_message: expandedMessage,
			message,
			notification_tag: notificationTag,
			title,
			item,
		} = remoteMessage?.data || {}

		// 메시지 데이터 처리 로직
		const bodyMessage = expandedMessage || message

		// Android에서 notifee를 사용한 고급 알림 표시
		const {
			default: notifee,
			AndroidColor,
			AndroidImportance,
			AndroidStyle,
			AndroidVisibility,
		} = notifeeData || {}

		const channelType = ['product', '상품 알람']
		const channelId = channelType[0]
		const channelName = channelType[1]
		const picture = imageUri || itemImageUri
		const hasImageUri = !!picture?.trim() && typeof picture === 'string'

		const style = {
			...(hasImageUri
				? {
						type: AndroidStyle.BIGPICTURE,
						picture: picture,
						title: title,
						summary: bodyMessage,
				  }
				: {
						type: AndroidStyle.BIGTEXT,
						text: bodyMessage,
				  }),
		}

		// Android 8.0+ 알림 채널 생성 (AOS 버전에 따라 다르게 사용함)
		if (Platform.Version >= 26) {
			if (typeof notifee?.createChannel === 'function') {
				await notifee.createChannel({
					id: channelId,
					lightColor: AndroidColor.RED,
					name: channelName,
					sound: 'default',
					vibration: true,
					vibrationPattern: [2, 300, 200, 300],
					visibility: AndroidVisibility.PUBLIC,
					importance: AndroidImportance.HIGH,
				})
			}
		}

		// notifee로 고급 알림 표시
		if (typeof notifee?.displayNotification === 'function') {
			await notifee.displayNotification({
				title: title,
				body: bodyMessage,
				android: {
					channelId,
					colorized: true,
					lights: [AndroidColor.RED, 300, 300],
					importance: AndroidImportance.HIGH,
					pressAction: {
						id: 'default',
					},
					sound: 'default',
					style: style,
					vibrationPattern: [2, 300, 200, 300],
					visibility: AndroidVisibility.PUBLIC,
				},
				data: remoteMessage?.data || {},
			})
		}
	} catch (error) {
		console.warn('onMessageReceived error >>', error)
		throw error
	}
}

/** Foreground 상태일 때 Firebase Message Handler 를 설정 */
export const setFirebaseForegroundMessageHandler = () => {
	return onMessage(messaging, onMessageReceived)
}

/** Foreground 상태일 때 Notifee Event 를 설정한다. */
export const setNotifeeForegroundEvent = () => {
	const { default: notifee } = notifeeData
	return notifee.onForegroundEvent(setNotifeeEventAction)
}
```

### 3. Background 메시지 처리

#### 앱이 백그라운드/종료 상태일 때 메시지 수신 (FCM)

```javascript
/** Background 상태일 때 Firebase Message Handler 를 설정 */
export const setFirebaseBackgroundMessageHandler = () => {
	messaging?.setBackgroundMessageHandler(async remoteMessage => {
		try {
			await onMessageReceived(remoteMessage)
			return Promise.resolve()
		} catch (error) {
			return Promise.resolve()
		}
	})
}
```

### 4. 메시지 타입별 처리

#### 데이터 전용 메시지 vs 알림 메시지

```javascript
// 서버에서 전송하는 메시지 구조 예시

// 1. 데이터 전용 메시지 (백그라운드에서도 핸들러 실행)
{
    "data": {
        "push_type": "ITEM",
        "view_type": "product",
        "landing_uri": "hsmoa://product/123",
        "title": "특가 상품 알림",
        "message": "지금 바로 확인하세요!",
        "item": "{\"id\":123,\"name\":\"상품명\"}"
    }
}

// 2. 알림 메시지 (시스템 알림 트레이에 자동 표시)
{
    "notification": {
        "title": "새로운 메시지",
        "body": "내용을 확인하세요"
    },
    "data": {
        "landing_uri": "myapp://home"
    }
}
```

---

## 실제 구현 패턴

### 1. 앱 초기화 시 FCM + Notifee 설정

```javascript
// App.js 또는 메인 컴포넌트에서
useEffect(() => {
	// Android에서 notifee 백그라운드 이벤트 설정
	if (Platform.OS === 'android') {
		// 백그라운드 메시지 핸들러 설정
		setFirebaseBackgroundMessageHandler()

		// 백그라운드 이벤트 설정
		setNotifeeBackgroundEvent()

		// 토큰 획득 및 서버 전송
		getPushTokenAsync().then(token => {
			if (token) {
				// 서버에 토큰 전송
				// updateDeviceInfo({ pushToken: token })
			}
		})
	}
}, [])
```

### 2. Foreground 메시지 처리 훅 (FCM + Notifee)

```javascript
// notificationUtil.js

/** 커스텀 훅으로 분리 */
export const useFCMForegroundHandler = () => {
    // Android 일 때 FCM, Notifee 설정
		useEffect(() => {
			const unsubscribeFirebase = setFirebaseForegroundMessageHandler()
			const unsubscribeNotifee = setNotifeeForegroundEvent()
			return () => {
				unsubscribeFirebase()
				unsubscribeNotifee()
			}
		}, [])
}

// App.js 에서 사용
function App() {
    useFCMForegroundHandler()
    return (
        // Display
    )
}
```

---

## 토큰 캐싱을 해볼까? (실제 적용 사례는 없지만...)

```javascript
import AsyncStorage from '@react-native-async-storage/async-storage'

const TOKEN_STORAGE_KEY = 'fcm_token'

const getCachedToken = async () => {
	try {
		return await AsyncStorage.getItem(TOKEN_STORAGE_KEY)
	} catch (error) {
		return null
	}
}

const setCachedToken = async token => {
	try {
		await AsyncStorage.setItem(TOKEN_STORAGE_KEY, token)
	} catch (error) {
		console.error('Token caching failed:', error)
	}
}
```

## 마무리

**FCM**과 **Notifee**를 푸시의 형태에 따라 서버에서 각각 나누어서 사용하도록 설계해놓았으니! 이제 안드로이드 푸시 구현이 완료되었다!

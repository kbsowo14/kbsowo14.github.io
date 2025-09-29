---
title: 'Expo Notifications 구현 가이드!'
date: 2025-05-29 16:00:00 +0900
category: [개발일지, React Native]
tags: [Expo, Notifications, 푸시알림, expo-notifications, 개발가이드]
---

Expo Notifications는 React Native 앱에서 알림/푸시 시스템을 쉽게 구현할 수 있게 해주는 라이브러리 입니다!
아래는 실제 프로젝트에서 사용중인 코드를 바탐으로 작업했던 구현 가이드입니다~ (코드로만 설명하겠다...)

---

## 설치 및 초기 설정

### 1. 패키지 설치

```bash
# Expo Notifications 설치
expo install expo-notifications

# 백그라운드 작업을 위한 TaskManager
expo install expo-task-manager

# 디바이스 정보 확인용
expo install expo-device
```

### 2. app.config.json 설정 (옵션은 선택사항)

```json
{
	"expo": {
		"ios": {
			"infoPlist": {
				"UIBackgroundModes": ["remote-notification"]
			}
		},
		"plugins": [
			[
				"expo-notifications",
				{
					"icon": "./local/assets/notification-icon.png",
					"color": "#ffffff",
					"sounds": ["./local/assets/notification-sound.wav"]
				}
			]
		]
	}
}
```

### 3. 기본 import 및 초기화

```javascript
// notificationUtil.js

import * as Notifications from 'expo-notifications'
import * as TaskManager from 'expo-task-manager'
import * as Device from 'expo-device'

// Android 8.0+ 에서는 알림 채널이 필요하지만
// expo-notifications가 자동으로 기본 채널을 생성해줍니다
// 커스텀 채널이 필요한 경우에만 추가 설정

// 커스텀 알림 채널 생성 (선택사항)
import * as Notifications from 'expo-notifications'

// 백그라운드 알림 처리를 위한 Task 이름
const BACKGROUND_NOTIFICATION_TASK = 'background-notification-task'

// Android에서 커스텀 채널 생성
if (Platform.OS === 'android') {
	await Notifications.setNotificationChannelAsync('default', {
		name: '기본 알림',
		importance: Notifications.AndroidImportance.HIGH,
		vibrationPattern: [0, 250, 250, 250],
		lightColor: '#FF231F7C',
		sound: 'default',
	})
}
```

---

## 핵심 구현 사항들

### 1. 기본 알림 설정

#### 알림 핸들러 설정

```javascript
// notificationUtil.js

/**
 * 기본 알람 설정을 한다. (iOS & Android 공통)
 *
 * 기본 설정 (알람 사운드, 배지, 얼럿 기능 설정)
 * 백그라운드일 때 알람 받기 설정
 */
export const initNotificationHandler = () => {
	try {
		// 공통 알림 핸들러 설정
		Notifications.setNotificationHandler({
			handleNotification: async () => ({
				shouldShowBanner: true, // 배너 표시
				shouldShowList: true, // 알림 목록에 표시
				shouldPlaySound: true, // 사운드 재생
				shouldSetBadge: true, // 앱 아이콘 배지 설정
			}),
		})

		/** 백그라운드에서 알람 받을 수 있게 TASK 등록하기 */
		TaskManager.defineTask(
			BACKGROUND_NOTIFICATION_TASK,
			({ data: notification, error, executionInfo }) => {
				// 백그라운드 알림 처리 로직
				updateAllStatisticActive(false)
				pushActionToStatistic(false, parseRawNotification(notification))
			}
		)
		Notifications.registerTaskAsync(BACKGROUND_NOTIFICATION_TASK)
	} catch (error) {
		console.warn('initNotificationHandler error >>>>', error)
	}
}
```

### 2. 권한 및 푸시 관리

#### 알림 권한 확인 및 요청

```javascript
// notificationUtil.js

/**
 * Notification 권한 승인인지 여부를 반환한다.
 *
 * @returns {Promise<boolean>} 권한 승인 상태이면 true / Denied, undetermined 이면 false
 */
export async function isNotificationsPermissionsGranted() {
	let isGranted = (await Notifications.getPermissionsAsync()).granted
	if (!isGranted) isGranted = (await Notifications.requestPermissionsAsync()).granted
	return await new Promise(resolve => resolve(isGranted))
}
```

#### 디바이스 푸시 토큰 획득

```javascript
// notificationUtil.js

/**
 * 기존 푸시코드를 삭제하고 새로운 푸시코드를 가져온다.
 *
 * @returns {Promise<string>} Push Token
 */
export async function getPushTokenAsync() {
	try {
		// 시뮬레이터일 경우 빈 문자열을 반환하고 함수 종료
		if (!Device.isDevice) {
			return '' // 시뮬레이터일 경우 빈 값을 반환
		}

		// 푸시 알림 등록 해제
		await Notifications.unregisterForNotificationsAsync()

		// 푸시 토큰 가져오기
		const pushToken = (await Notifications.getDevicePushTokenAsync()).data
		if (pushToken) {
			return pushToken
		}

		return '' // 토큰을 받지 못한 경우 빈 문자열 반환
	} catch (error) {
		console.error('Error fetching push token:', error)
		return '' // 오류 발생 시 빈 문자열 반환
	}
}
```

#### 권한 및 푸시 토큰 관리 로직 생성 (재시도 로직 포함)

```javascript
// notificationUtil.js

/**
 * 푸시 토큰을 가져와서 갱신해주고 이후의 프로세스까지 실행해준다.
 *
 * @property {boolean} isRetry 실패시 재실행할지 여부
 * @property {function} onDeniedPermission 권한 획득 실패할 때 실행할 함수
 * @property {function} onGrantedPermission 권한 획득 성공했을 때 실행할 함수
 */
export async function awaitPushToken({
	isRetry = true,
	onDeniedPermission = () => {},
	onGrantedPermission = () => {},
}) {
	const MAX_RETRY_COUNT = 10

	if (awaitPushTokenRetryTimer) clearTimeout(awaitPushTokenRetryTimer)

	const run = async (retryCount = 1) => {
		if (retryCount > MAX_RETRY_COUNT) return

		try {
			if (!(await isNotificationsPermissionsGranted())) {
				if (isRetry) awaitPushTokenRetryTimer = setTimeout(() => run(retryCount + 1), 2500) // 2.5초 뒤에 재시도
				if (retryCount === MAX_RETRY_COUNT - 1) onDeniedPermission()
			} else {
				const pushToken = await getPushTokenAsync()
				if (pushToken) updateDeviceInfo({ pushToken: pushToken })
				onGrantedPermission()
			}
		} catch (error) {}
	}

	run()
}
```

### 3. Foreground 알림 처리

#### Foreground 알림 리스너

```javascript
// notificationUtil.js

/**
 * Foreground 에서의 알람을 설정한다.
 *
 * @description iOS & Android 공통 expo-notifications 사용
 */
export const setForegroundNotificationHandler = () => {
	useEffect(() => {
		let isMounted = true

		// 앱이 켜져 있는데, 푸시가 옴 (iOS & Android 공통)
		const notificationListener = Notifications.addNotificationReceivedListener(notification => {
			pushActionToStatistic(false, parseRawNotification(notification))
		})

		// 푸시를 유저가 누름 (iOS & Android 공통)
		const notiResponseListener = Notifications.addNotificationResponseReceivedListener(
			async response => {
				if (!response?.notification) return
				setPushClickEvent(parseRawNotification(response?.notification))
			}
		)

		// 앱 시작 시 마지막 알림 응답도 확인 (iOS & Android 공통)
		Notifications.getLastNotificationResponseAsync().then(response => {
			if (!response?.notification || !isMounted) return
			setPushClickEvent(parseRawNotification(response?.notification))
		})

		return () => {
			isMounted = false
			notificationListener?.remove()
			notiResponseListener?.remove()
		}
	}, [])
}
```

**리스너 종류**

- `addNotificationReceivedListener`: 알림 수신 시 (표시만, 클릭 아님) - iOS & Android 공통
- `addNotificationResponseReceivedListener`: 알림 클릭 시 - iOS & Android 공통
- `getLastNotificationResponseAsync`: 앱 시작 시 마지막 알림 응답 확인 - iOS & Android 공통

#### 플랫폼별 알림 데이터 구조 통합 (혹시 모르니 데이터를 직접 확인해보세요!)

```javascript
/** 푸시 원본 데이터를 파싱한다. */
export const parseRawNotification = rawData => {
	let data
	if (isAndroid) {
		// Android 일 때 foreground || background 형식
		data = rawData?.request?.trigger?.remoteMessage || rawData?.notification
	} else {
		// iOS 일 때 foreground || background 형식
		data = rawData?.request?.trigger?.payload || rawData
	}
	return data
}
```

### 4. 로컬 알림 생성 및 예약 (프로젝트에는 사용하지 않았지만 일단 기본적인 구현 방법 기록...)

```javascript
// 기본 로컬 알림
await Notifications.scheduleNotificationAsync({
	content: {
		title: '알림 제목',
		body: '알림 내용',
		data: { customData: '커스텀 데이터' },
	},
	trigger: null, // 즉시 표시
})

// 지연 알림 (5초 후)
await Notifications.scheduleNotificationAsync({
	content: {
		title: '지연된 알림',
		body: '5초 후에 표시됩니다',
	},
	trigger: { seconds: 5 },
})

// 반복 알림 (매일 오전 9시)
await Notifications.scheduleNotificationAsync({
	content: {
		title: '일일 리마인더',
		body: '오늘도 화이팅!',
	},
	trigger: {
		hour: 9,
		minute: 0,
		repeats: true,
	},
})
```

### 5. React Hook으로 알림 관리 (통합 알림 관리 훅)

```javascript
export const useNotifications = () => {
	const [notifications, setNotifications] = useState([])
	const [hasPermission, setHasPermission] = useState(false)

	useEffect(() => {
		// 초기화
		const initialize = async () => {
			const permission = await isNotificationsPermissionsGranted()
			setHasPermission(permission)

			if (permission) {
				await notificationService.initialize()
			}
		}

		initialize()
	}, [])

	useEffect(() => {
		if (!hasPermission) return

		// 알림 리스너 설정
		const receivedListener = Notifications.addNotificationReceivedListener(notification => {
			setNotifications(prev => [...prev, notification])
		})

		const responseListener = Notifications.addNotificationResponseReceivedListener(response => {
			// 알림 클릭 처리 (handleNotificationClick 함수에서 내부 동작 로직을 구현하면 됨!)
			handleNotificationClick(response.notification)
		})

		return () => {
			receivedListener.remove()
			responseListener.remove()
		}
	}, [hasPermission])

	const scheduleNotification = useCallback(
		async (title, body, trigger) => {
			if (!hasPermission) {
				console.warn('No notification permission')
				return null
			}

			return await notificationService.scheduleLocalNotification(title, body, {}, trigger)
		},
		[hasPermission]
	)

	return {
		notifications,
		hasPermission,
		scheduleNotification,
		cancelNotification: notificationService.cancelNotification,
	}
}
```

---

## 신경써야되는 문제!

### 1. 포커싱 될때마다 권한 상태 확인 (프로젝트 성격마다 다름 : 푸시가 필수인 서비스일때만 필요)

```javascript
// 앱 포커스 시마다 권한 상태 재확인
useEffect(() => {
	const handleAppStateChange = async nextAppState => {
		if (nextAppState === 'active') {
			const { status } = await Notifications.getPermissionsAsync()
			setHasPermission(status === 'granted')
		}
	}

	const subscription = AppState.addEventListener('change', handleAppStateChange)
	return () => subscription?.remove()
}, [])
```

### 2. 백그라운드 로직은 가볍게 사용하기 (에셋 갱신이나 무거운 로직은 지양하기)

```javascript
// 백그라운드에서는 제한된 작업만 수행
TaskManager.defineTask(BACKGROUND_NOTIFICATION_TASK, ({ data, error }) => {
	if (error) {
		console.error('Background task error:', error)
		return
	}

	try {
		// 최소한의 처리만 수행
		const notification = parseRawNotification(data)

		// 간단한 통계 전송
		sendQuickStatistics(notification)

		// 로컬 스토리지에 저장 (다음 앱 실행 시 처리)
		storeNotificationForLater(notification)
	} catch (error) {
		console.error('Background processing error:', error)
	}
})
```

## 마무리

Expo Notifications로 **iOS와 Android 모두 동일한 API**를 통해 알림 시스템을 구축~~ 완료!!

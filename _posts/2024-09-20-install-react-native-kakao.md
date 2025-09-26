---
title: React Native Kakao 라이브러리 등장! - 로그인부터 공유까지 구현하기
date: 2024-09-20 16:00:00 +0900
category: [개발일지, React Native]
tags: [React Native, 카카오SDK, 카카오로그인, 카카오공유, 소셜로그인, 게임화, 개발일지]
---

# React Native Kakao 라이브러리 등장! - 로그인부터 공유까지 구현하기 🚀

모바일 앱에서 **카카오톡 로그인**과 **공유 기능**은 이제 필수가 되었다.
기존에 공식 라이브러리가 제공되지않은 탓에 웹뷰로 우회해서 웹으로 로그인 성공시켜 토큰을 받아 로그인 시키고 카카오 공유도 헤더를 모바일로 바꿔서 모바일 공유 UI로 나오도록 페이크를 주어서 구현했던 것을, 이제 React Native 카카오 공식 라이브러리가 등장한 참에 해당 라이브러리로 교체를 해보고자 한다.

## 📋 개요

### 구현 목표

- **카카오톡 로그인**: 간편한 소셜 로그인 구현
- **카카오톡 공유**: 게임 결과 및 초대 링크 공유
- **서버 연동**: 공유 성과 추적 및 보상 지급

### 사용 라이브러리

```json
{
	"@react-native-kakao/core": "^2.x.x",
	"@react-native-kakao/user": "^2.x.x",
	"@react-native-kakao/share": "^2.x.x"
}
```

## 🛠 카카오 SDK 초기 설정

### 1. expo-build-properties 설정 추가

```javascript
'@react-native-kakao/core',
    {
      nativeAppKey:
        APP_ENV === 'development'
          ? '27xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
          : '88xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx',
      android: {
        forwardKakaoLinkIntentFilterToMainActivity: true,
        authCodeHandlerActivity: true, // 필수 카카오톡 인증 수신 vip백도어
      },
      ios: {
        handleKakaoOpenUrl: true, // 필수 카카오톡 인증 수신 vip백도어
      },
    },
```

### 2. 기본 SDK 초기화

initializeKakaoSDK 메소드로 앱 시작시 초기화 해주기!
마침 우리 앱은 개발용 앱 계정과 실사용 앱 계정을 다르게 관리하고 있어서 isEnvDev 메소드로 개발용과 실사용을 구분해서 초기화 해주었다.

```javascript
import { initializeKakaoSDK } from '@react-native-kakao/core'

// App.js에서 초기화
useEffect(() => {
	// 환경별 앱 키 설정
	const kakaoAppKey = isEnvDev()
		? '27xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx' // 개발용
		: '88xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx' // 프로덕션용

	initializeKakaoSDK(kakaoAppKey)
}, [])
```

## 🔐 카카오톡 로그인 구현을 해보자!

### 1. 로그인 상태 관리

AuthContext에서 카카오 로그인을 통합 관리:

```javascript
import { isLogined, logout as kakaoLogout } from '@react-native-kakao/user'

const logout = async ({ frame, isStat = false } = {}) => {
	try {
    // 기존 로그아웃 로직들...

		// 카카오 로그인 상태 확인 및 로그아웃
		try {
			const isKakaoLogined = await isLogined()
			if (isKakaoLogined) {
				await kakaoLogout()
			}
		} catch (e) {
			console.warn('error in logout AuthContext', e)
      // 필요시 로그아웃 통계로 모니터링 해주기
		}

		// 기존 로그아웃 로직들...
	} catch (error) {
		console.warn('error in logout AuthContext', error)
	}
}
```

### 2. 로그인 플로우 통합

카카오 로그인을 기존 인증 시스템과 매끄럽게 연결:

```javascript
const handleKakaoLogin = async () => {
	try {
		// 카카오 로그인 실행
		const kakaoUserInfo = await kakaoLogin()

		// 서버에 카카오 사용자 정보 전송
		const serverResponse = await api.auth.loginWithKakao[1]({
			kakao_id: kakaoUserInfo.id,
			kakao_email: kakaoUserInfo.kakao_account?.email,
			kakao_nickname: kakaoUserInfo.kakao_account?.profile?.nickname,
      access_token: kakaoUserInfo.access_token,
		})

		// 기존 로그인 로직들...
	} catch (error) {
		console.error('카카오 로그인 실패:', error)
		setToast({
			visible: true,
			content: '카카오 로그인에 실패했습니다. 다시 시도해주세요.',
		})
	}
}
```

## 📤 카카오톡 공유 시스템을 구현 해보자!

### 1. 커스텀 템플릿을 이용해서 공유하기

```javascript
import { shareCustomTemplate } from '@react-native-kakao/share'

const shareMission = async () => {
	try {
		// 초대 링크를 생성해서 가져오는 로직들... (부디베이스로 마케터가 이미지와 멘트를 설정 가능)

		// 부디베이스에서 공유 콘텐츠 데이터 조회 (react-query 사용)
		const shareContentData = await queryClient.fetchQuery({
			queryKey: api.manager.getShareContent[0](),
			queryFn: () =>
				api.manager.getShareContent[1]({
					env: getEnv(),
					uid: 'share',
				}),
		})

		const {
			title = '기본 제목',
			description = '기본 설명\n',
		} = shareContentData?.[0] || {}

		// 카카오톡 커스텀 템플릿으로 공유
		const shareResult = await shareCustomTemplate({
			templateId: 000000, // 카카오 디벨로퍼스에서 등록한 템플릿 ID
			useWebBrowserIfKakaoTalkNotAvailable: true,
			templateArgs: {
				title: title,
				description: description,
				mobileUrl: url,
				desktopUrl: url,
				buttonTitle: '확인하기',
				imageUrl: gameResultImageUrl,
			},
			serverCallbackArgs: { // 공유 성공 시 콜백용 데이터 추가 (프로젝트 한정)
				user_token: userInfo?.user_token,
				service: 'share',
				share_type: `game-api:share:${mission?.id}`,
			},
		})
	} catch (error) {
		console.error('공유 실패:', error)
		setToast({
			visible: true,
			content: '공유에 실패했습니다. 다시 시도해주세요.',
		})
	}
}
```

### 2. 공유하면 미션의 기회를 더 주는 시스템을 구현!

룰렛 게임과 연동된 공유 인센티브 시스템을 구현해보자!

```javascript
	// 카카오 공유 가능 횟수 조회
	const { data: kakaoShareAvailableInfo } = useQuery({
		queryKey: api.mission.getKakaoShareAvailableInfo[0](),
		queryFn: () =>
			api.mission.getKakaoShareAvailableInfo[1]({
				kakao_share_config_id: kakao_share_config_id,
			}),
		enabled: !!kakao_share_config_id,
	})

	const { max_share_count = 0, remaining_share_count = 0 } = kakaoShareAvailableInfo || {}

	const handleShareForRoulette = async () => {
		if (!remaining_share_count) {
			setToast({
				visible: true,
				content: '오늘 공유 횟수를 모두 사용했어요.',
			})
			return
		}

		// 공유 확인 알럿 (alert 내부 공용 컴포넌트 사용)
		showAlert({
			visible: true,
			content: (
				<View className="justify-center items-center mt-2">
					<Text>공유 하고 기회 더받기!</Text>
				</View>
			),
			okText: '공유하기',
			cancelText: '뒤로가기',
			onOk: () => {
				shareMission() // 위에 설계해놓은 공유 함수 동작
			},
		})
	}
```

## 📈 성과 및 효과

### **사용자 유입 증가**
### **공유 참여율 증가**
### **재방문율 증가**

## 🔚 마무리

react-native-kakao 라이브러리 감사합니다!

---

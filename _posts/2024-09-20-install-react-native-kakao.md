---
title: React Native Kakao 라이브러리 등장! - 로그인부터 공유까지 구현하기
date: 2024-09-20 16:00:00 +0900
category: [개발일지, React Native]
tags: [React Native, 카카오SDK, 카카오로그인, 카카오공유, 소셜로그인, 게임화, 개발일지]
---

모바일 앱에서 카카오톡 로그인과 공유 기능은 이제 필수가 되었다.

당연히 우리의 앱에도 카카오 로그인과 카카오 공유 시스템은 존재했다.
물론 Expo RN 프로젝트에 공식 지원하는 라이브러리가 없었던 터라 WebView로 웹 카카오 로그인과 공유 시스템을 열어주고 header에 모바일인 것처럼 속여 모바일 카카오 로그인과 공유인 것처럼 Fake로 구현했다.
그로인해 정말 알기 힘든 유저 패턴과 환경을 예상하면서 예방로직을 넣어두었다.
("카카오앱이 없거나 문제가 있는 유저는 어떡하지?" > 카카오앱이 없으면 공유 시나리오가 멈출테니까 웹뷰 반응이 3초~5초동안 없으면 그냥 다시 돌려보낼까?)

이런 좋지않은 로직들을 덕지덕지 넣으며 유지보수를 걱정하고있었지만
마침 카카오가 인정한 공식 Expo RN KAKAO 라이브러리가 등장했다!

기존에 공식 라이브러리가 제공되지않은 탓에 웹뷰로 우회해서 웹으로 로그인 성공시켜 토큰을 받아 동작시키던 코드들도 이제 간단한 메소드 한줄로 안녕이다!
바로 프로젝트에 채용해보자!

---

## 라이브러리

```json
{
	"@react-native-kakao/core": "^2.x.x",
	"@react-native-kakao/user": "^2.x.x",
	"@react-native-kakao/share": "^2.x.x"
}
```

---

## 카카오 SDK 초기 설정

### 1. expo-build-properties 설정 추가

당연히 Expo 클라우드 빌드 시 함께 빌드되도록 빌드옵션을 조정해준다.

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

---

## 카카오톡 로그인 구현을 해보자!

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

카카오 로그인을 기존 인증 시스템과 매끄럽게 연결해준다.

```javascript
import { login as kakaoLogin, me } from '@react-native-kakao/user'

const handleKakaoLogin = async () => {
	try {
		// 카카오 로그인 실행
		const credential = await kakaoLogin({})

		// 카카오 사용자 정보 가져오기
		const profileInfo = await me()

		// 카카오 사용자 정보 저장 해놓기
		await SecureStore.setItemAsync(KAKAO_ID, JSON.stringify(profileInfo?.id))

		// 서버에 카카오 사용자 정보 전송 (서버 요구 형태로 전달)
		// const loginData = {
		// 	linked_with: 'kakao',
		// 	email: profileInfo?.email || '',
		// 	sns_id: profileInfo?.id || '',
		// 	code: credential?.accessToken || '',
		// }

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

---

## 카카오톡 공유 시스템을 구현 해보자!

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

## 마무리

react-native-kakao 감사합니다!

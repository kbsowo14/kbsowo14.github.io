---
title: 'Expo 프로젝트에 서드파티 네이티브 SDK 모듈 만들기 - 버즈빌 베네핏허브 연동'
date: 2025-08-20 17:00:00 +09:00
category: [개발일지, React Native]
tags: [Expo, 네이티브모듈, SDK통합, Kotlin, Swift, 서드파티, 버즈빌]
---

Expo로 개발하다 보면 가끔 이런 상황을 꽤 자주 마주하게 된다!
"Expo 설정이 없네?"

이러면 이제 Android & iOS 네이티브 모듈을 따로 만들어서 Expo 빌드와 연결해주어야 한다!

마침 버즈빌 업체의 버즈베네핏 SDK 설치 요청이 들어왔는데, 역시 네이티브 모듈을 패키징 해야했다.
(**Android(Kotlin)** 과 **iOS(Swift)** 네이티브 SDK뿐이었거든요...)

> 참고자료: [Expo 공식 문서 - Third Party Library 모듈 만들기](https://docs.expo.dev/modules/third-party-library/)

---

## 프로젝트 구조 살펴보기

```
/modules/buzzvil-module/
├── android/                                        # Android 네이티브 구현
│   ├── build.gradle                                # Android 빌드 설정 파일
│   └── src/main/java/expo/modules/buzzvilmodule/
│       └── BuzzvilModule.kt                        # Kotlin으로 구현한 메인 모듈 (중요)
├── ios/                                            # iOS 네이티브 구현
│   ├── BuzzvilModule.podspec                       # iOS 의존성 패키지 설정 파일
│   └── BuzzvilModule.swift                         # Swift로 구현한 메인 모듈 (중요)
├── expo-module.config.json                         # Expo 모듈 설정 파일
└── index.ts                                        # TypeScript 인터페이스 도출 파일
```

---

## 모듈 생성하기

```bash
npx create-expo-module --local buzzvil-module
```

완료되면 modules 폴더와 buzzvil-module 폴더가 생성되면서 내부의 기본 뼈대가될 파일들이 자동으로 생성되어있을 겁니다!
View 컴포넌트를 만들어서 사용할 것은 아니기때문에 View 와 관련된 파일들은 깔끔하게 제거를 해줬습니다! (버즈베네핏 SDK 내부에서 View를 다 동작시켜주거든요.)

---

## 인터페이스 정의

이제 네이티브 코드들을 우리 RN 프로젝트에서 사용할 수 있도록 export 함수들을 정의해줍니다!

**`index.ts`**

```typescript
import { requireNativeModule } from 'expo-modules-core'

const BuzzvilModule = requireNativeModule('BuzzvilModule')

export default {
	// SDK 초기화
	initialize: (appId: string): Promise<any> => {
		return new Promise((resolve, reject) => {
			if (typeof BuzzvilModule?.initialize !== 'function' || !appId) {
				reject(new Error('initialize 메서드를 사용할 수 없습니다'))
				return
			}
			BuzzvilModule.initialize(appId)
				.then((result: any) => resolve(result))
				.catch((error: any) => reject(error))
		})
	},

	// 사용자 로그인
	login: (
		userId: string,
		options: { gender?: string; birthYear?: number } = {}
	): Promise<any> => {
		return new Promise((resolve, reject) => {
			if (typeof BuzzvilModule?.login !== 'function') {
				reject(new Error('login 메서드를 사용할 수 없습니다'))
				return
			}
			BuzzvilModule.login(userId, options.gender, options.birthYear)
				.then((result: any) => resolve(result))
				.catch((error: any) => reject(error))
		})
	},

	// 로그아웃
	logout: () => {
		if (typeof BuzzvilModule?.logout !== 'function') {
			return
		}
		BuzzvilModule.logout()
	},

	// 로그인 상태 확인
	isLoggedIn: () => {
		if (typeof BuzzvilModule?.isLoggedIn !== 'function') {
			return null
		}
		return BuzzvilModule.isLoggedIn()
	},

	// 베네핏허브 화면 표시
	show: (): Promise<any> => {
		return new Promise((resolve, reject) => {
			if (typeof BuzzvilModule?.show !== 'function') {
				reject(new Error('베네핏허브 show 메서드를 사용할 수 없습니다'))
				return
			}
			BuzzvilModule.show()
				.then((result: any) => resolve(result))
				.catch((error: any) => reject(error))
		})
	},
}
```

여기서 중요한 건 **에러 처리**입니다. 네이티브 모듈이 제대로 로드되지 않았을 때를 대비해서 안전장치를 만들어뒀습니다!

---

## Android 네이티브 모듈 구현

연결해주는 인터페이스를 정의했으니, 인터페이스 코드에서 꺼내서 사용할 실제 안드로이드 네이티브 코드들을 구현해줍니다!
제가 다 작성한건아니고, 모듈을 처음 create하면 일부분은 알아서 작성되어져 있습니다! 몇가지만 수정해놓으면 되요!
(여기서 중요한건! 반드시 버즈베네핏 공식 문서 가이드를 참고해서 구현해줍니다!)

### Android 빌드 설정

**`android/build.gradle`**

```javascript
apply plugin: 'com.android.library'
apply plugin: 'kotlin-android'

group = 'expo.modules.buzzvilmodule'
version = '0.6.3'

// Expo 모듈 설정
def expoModulesCorePlugin = new File(project(":expo-modules-core").projectDir.absolutePath, "ExpoModulesCorePlugin.gradle")
apply from: expoModulesCorePlugin
applyKotlinExpoModulesCorePlugin()
useCoreDependencies()
useExpoPublishing()

android {
  namespace "expo.modules.buzzvilmodule"
  defaultConfig {
    versionCode 1
    versionName "0.6.3"
  }

  lintOptions {
    abortOnError false
  }
}

// 버즈베네핏 SDK 저장소 추가
repositories {
  google()
  mavenCentral()
  maven { url "https://dl.buzzvil.com/public/maven" }
}

dependencies {
  // Expo 모듈 의존성
  implementation project(':expo-modules-core')

  // React Native 의존성
  implementation 'com.facebook.react:react-native:+'

  // Kotlin 표준 라이브러리
  implementation 'org.jetbrains.kotlin:kotlin-stdlib:1.8.10'

  // 버즈빌 SDK
  def buzzvilBomVersion = "6.0.0"
  implementation(platform("com.buzzvil:buzzvil-bom:$buzzvilBomVersion")) {
    exclude(group: 'com.buzzvil', module: 'buzz-covi')
  }
  implementation("com.buzzvil:buzzvil-sdk") {
    exclude(group: 'com.buzzvil', module: 'buzz-covi')
  }
}
```

### Kotlin 메인 모듈 구현

이제 나는 잠시 안드로이드 네이티브 개발자가 되어야 한다! 허허허허허허

**`android/src/main/java/expo/modules/buzzvilmodule/BuzzvilModule.kt`**

```kotlin
package expo.modules.buzzvilmodule

import android.util.Log
import expo.modules.kotlin.modules.Module
import expo.modules.kotlin.modules.ModuleDefinition
import expo.modules.kotlin.Promise

import android.app.Application
import com.buzzvil.buzzbenefit.BuzzBenefitConfig
import com.buzzvil.buzzbenefit.benefithub.BuzzBenefitHub
import com.buzzvil.sdk.BuzzvilSdk
import com.buzzvil.sdk.BuzzvilSdkUser
import com.buzzvil.sdk.BuzzvilSdkLoginListener
import com.buzzvil.sdk.BuzzvilSdkLoginListener.ErrorType
import org.json.JSONObject

class BuzzvilModule : Module() {
    override fun definition() = ModuleDefinition {

        Name("BuzzvilModule")

        // SDK 초기화
        AsyncFunction("initialize") { appId: String, promise: Promise ->
            val context = appContext.reactContext ?: run {
                promise.reject("NO_CONTEXT", "리액트 컨텍스트를 찾을 수 없습니다", null)
                return@AsyncFunction
            }

            try {
                val buzzBenefitConfig = BuzzBenefitConfig.Builder(appId)
                    .build()

                BuzzvilSdk.initialize(
                    application = context.applicationContext as Application,
                    buzzBenefitConfig = buzzBenefitConfig
                )

                Log.d("BuzzvilModule", "Buzzvil SDK 초기화 성공")

                val result = JSONObject()
                result.put("success", true)
                result.put("message", "초기화 성공")
                promise.resolve(result.toString())
            } catch (e: Exception) {
                Log.e("BuzzvilModule", "Buzzvil SDK 초기화 실패: ${e.message}")
                promise.reject("INIT_ERROR", "초기화 실패: ${e.message}", e)
            }
        }

        // 사용자 로그인
        AsyncFunction("login") { userId: String, gender: String?, birthYear: Int?, promise: Promise ->
            val context = appContext.reactContext ?: run {
                promise.reject("NO_CONTEXT", "리액트 컨텍스트를 찾을 수 없습니다", null)
                return@AsyncFunction
            }

            try {
                val buzzvilSdkUser = BuzzvilSdkUser(
                    userId = userId,
                    gender = gender?.let { BuzzvilSdkUser.Gender.valueOf(it) } ?: BuzzvilSdkUser.Gender.UNKNOWN,
                    birthYear = birthYear
                )

                BuzzvilSdk.login(
                    buzzvilSdkUser = buzzvilSdkUser,
                    listener = object : BuzzvilSdkLoginListener {
                        override fun onSuccess() {
                            Log.d("BuzzvilModule", "로그인 성공: $userId")
                            val result = JSONObject()
                            result.put("success", true)
                            result.put("message", "로그인 성공")
                            promise.resolve(result.toString())
                        }

                        override fun onFailure(errorType: ErrorType) {
                            Log.e("BuzzvilModule", "로그인 실패: $errorType")
                            promise.reject("LOGIN_ERROR", "로그인 실패: $errorType", null)
                        }
                    }
                )
            } catch (e: Exception) {
                Log.e("BuzzvilModule", "로그인 오류: ${e.message}")
                promise.reject("LOGIN_ERROR", "로그인 오류: ${e.message}", e)
            }
        }

        // 로그아웃
        Function("logout") {
            try {
                BuzzvilSdk.logout()
                Log.d("BuzzvilModule", "로그아웃 성공")
                return@Function true
            } catch (e: Exception) {
                Log.e("BuzzvilModule", "로그아웃 오류: ${e.message}")
                return@Function false
            }
        }

        // 로그인 상태 확인
        Function("isLoggedIn") {
            try {
                val isLoggedIn = BuzzvilSdk.isLoggedIn
                Log.d("BuzzvilModule", "로그인 상태: $isLoggedIn")
                return@Function isLoggedIn
            } catch (e: Exception) {
                Log.e("BuzzvilModule", "로그인 상태 확인 오류: ${e.message}")
                return@Function false
            }
        }

        // 베네핏허브 화면 표시 (가장 복잡한 부분!)
        AsyncFunction("show") { promise: Promise ->
            try {
                val activity = appContext.activityProvider?.currentActivity
                    ?: run {
                        promise.reject("NO_ACTIVITY", "액티비티를 찾을 수 없습니다", null)
                        return@AsyncFunction
                    }

                activity.runOnUiThread {
                    try {
                        var currentActivity = activity
                        if (activity.isFinishing) {
                            Log.d("BuzzvilModule", "현재 액티비티가 종료 중입니다. 유효한 액티비티를 찾는 중...")
                            currentActivity = appContext.activityProvider?.currentActivity
                                ?: run {
                                    promise.reject("NO_VALID_ACTIVITY", "유효한 액티비티를 찾을 수 없습니다", null)
                                    return@runOnUiThread
                                }
                        }

                        if (!currentActivity.isFinishing && !currentActivity.isDestroyed) {
                            BuzzBenefitHub.show(currentActivity)
                            Log.d("BuzzvilModule", "베네핏허브가 성공적으로 표시되었습니다")

                            val result = JSONObject()
                            result.put("success", true)
                            result.put("message", "베네핏허브 표시 성공")
                            promise.resolve(result.toString())
                        } else {
                            promise.reject("INVALID_ACTIVITY_STATE", "액티비티가 유효한 상태가 아닙니다", null)
                        }
                    } catch (e: Exception) {
                        Log.e("BuzzvilModule", "UI 스레드에서 베네핏허브 표시 중 오류 발생: ${e.message}")
                        promise.reject("SHOW_ERROR", "베네핏허브 표시 중 오류 발생: ${e.message}", e)
                    }
                }
            } catch (e: Exception) {
                Log.e("BuzzvilModule", "베네핏허브 표시 중 오류 발생: ${e.message}")
                promise.reject("SHOW_ERROR", "베네핏허브 표시 중 오류 발생: ${e.message}", e)
            }
        }
    }
}
```

---

## iOS 네이티브 모듈 구현

### iOS 의존성 설정

이제 iOS 차례다... Podfile에 의존성을 추가해줍니다! (여기도 공식 문서를 반드시 참고해야합니다!)

**`ios/BuzzvilModule.podspec`**

```ruby
Pod::Spec.new do |s|
  s.name           = 'BuzzvilModule'
  s.version        = '1.0.0'
  s.summary        = 'Buzzvil SDK Module for Expo'
  s.description    = 'Expo module for integrating Buzzvil SDK'
  s.author         = 'hsmoa'
  s.homepage       = 'https://docs.expo.dev/modules/'
  s.platforms      = {
    :ios => '15.1',
    :tvos => '15.1'
  }
  s.source         = { git: '' }
  s.static_framework = true

  # 핵심 의존성들 (버즈베네핏 SDK)
  s.dependency 'ExpoModulesCore'
  s.dependency 'BuzzvilSDK', '~> 6.0.0'

  # Swift/Objective-C 호환성
  s.pod_target_xcconfig = {
    'DEFINES_MODULE' => 'YES',
    'SWIFT_VERSION' => '5.0'
  }

  s.source_files = "**/*.{h,m,mm,swift,hpp,cpp}"
end
```

### Swift 메인 모듈 구현

또다시 나는 잠시 iOS 네이티브 개발자가 되어야 한다! 허허허허허허

**`ios/BuzzvilModule.swift`**

```swift
import ExpoModulesCore
import BuzzvilSDK

public class BuzzvilModule: Module {
  public func definition() -> ModuleDefinition {

    Name("BuzzvilModule")

    // SDK 초기화
    AsyncFunction("initialize") { (appId: String, promise: Promise) in
      do {
        let config = BuzzBenefitConfig.Builder(appId: appId)
            .build()
        BuzzBenefit.shared.initialize(with: config)
        promise.resolve(["success": true, "message": "초기화 성공"])
      } catch {
        promise.reject(error)
      }
    }

    // 사용자 로그인
    AsyncFunction("login") { (userId: String, gender: String?, birthYear: Int?, promise: Promise) in
      let userBuilder = BuzzBenefitUser.Builder(userId: userId)

      // 성별 (Optional)
      if let gender = gender {
        switch gender.lowercased() {
        case "male":
          userBuilder.setGender(.male)
        case "female":
          userBuilder.setGender(.female)
        default:
          break
        }
      }

      // 생년 (Optional)
      if let birthYear = birthYear {
        userBuilder.setBirthYear(birthYear)
      }

      let buzzBenefitUser = userBuilder.build()

      BuzzBenefit.shared.login(
        with: buzzBenefitUser,
        onSuccess: {
          print("버즈빌 로그인 성공")
          promise.resolve(["success": true, "message": "로그인 성공"])
        },
        onFailure: { error in
          print("버즈빌 로그인 실패: \(error.localizedDescription)")
          promise.reject(error)
        }
      )
    }

    // 로그아웃
    Function("logout") {
      BuzzBenefit.shared.logout()
    }

    // 로그인 상태 확인
    Function("isLoggedIn") { () -> Bool in
      return BuzzBenefit.shared.isLoggedIn()
    }

    // 베네핏허브 화면 표시
    AsyncFunction("show") { (promise: Promise) in
      DispatchQueue.main.async {
        guard let rootViewController = UIApplication.shared.windows.first?.rootViewController else {
          promise.reject(NSError(domain: "BuzzvilModule", code: 1001, userInfo: [NSLocalizedDescriptionKey: "최상위 뷰 컨트롤러를 찾을 수 없습니다"]))
          return
        }

        // 최상위 뷰 컨트롤러 찾는 재귀 함수
        func topViewController(controller: UIViewController? = rootViewController) -> UIViewController? {
          if let navigationController = controller as? UINavigationController {
            return topViewController(controller: navigationController.visibleViewController)
          }
          if let tabController = controller as? UITabBarController {
            if let selected = tabController.selectedViewController {
              return topViewController(controller: selected)
            }
          }
          if let presented = controller?.presentedViewController {
            return topViewController(controller: presented)
          }
          return controller
        }

        guard let topVC = topViewController() else {
          promise.reject(NSError(domain: "BuzzvilModule", code: 1002, userInfo: [NSLocalizedDescriptionKey: "최상위 뷰 컨트롤러를 찾을 수 없습니다"]))
          return
        }

        do {
          // 인스턴스 생성 후 show 메서드 호출
          let benefitHub = BuzzBenefitHub()
          benefitHub.show(on: topVC)
          print("베네핏허브가 성공적으로 표시되었습니다")
          promise.resolve(["success": true, "message": "베네핏허브 표시 성공"])
        } catch {
          print("베네핏허브 표시 중 오류 발생: \(error.localizedDescription)")
          promise.reject(error)
        }
      }
    }
  }
}
```

---

## 프로젝트 설정 파일 수정

자이제 모듈 설치는 끝났다! (빌드하고 에러나고 빌드하고 에러나고... 를 수없이 반복한 후 결국 해냈다!)
이제 만든 모듈을 프로젝트에서 사용할 수 있도록 설정해야 합니다!

### TypeScript 경로 설정

**`tsconfig.json`**

```json
{
	"extends": "expo/tsconfig.base",
	"compilerOptions": {
		"baseUrl": "./src",
		"paths": {
			"react": ["./node_modules/@types/react"],
			"@buzzvil-module/*": ["./modules/buzzvil-module/*"] // 모듈 경로 추가
		},
		"module": "ESNext"
	}
}
```

### JavaScript 경로 설정

**`jsconfig.json`**

```json
{
	"compilerOptions": {
		"jsx": "react",
		"baseUrl": "./src",
		"paths": {
			"@buzzvil-module/*": ["./modules/buzzvil-module/*"] // 모듈 경로 추가
		}
	},
	"include": ["./src", "./*"],
	"exclude": ["node_modules"]
}
```

### Babel 모듈 해석 설정

**`babel.config.js`**

```javascript
module.exports = function (api) {
	api.cache(true)
	return {
		presets: ['babel-preset-expo'],
		plugins: [
			'nativewind/babel',
			[
				'module-resolver',
				{
					root: ['./src'],
					alias: {
						'@buzzvil-module': './modules/buzzvil-module', // 모듈 별칭 추가
					},
				},
			],
			'react-native-reanimated/plugin',
		],
		env: {
			production: {
				plugins: ['transform-remove-console'],
			},
		},
	}
}
```

### Expo 빌드 설정에 저장소 추가

**`app.config.js`**

```javascript
// ... 기존 설정들

plugins: [
	// ... 기존 플러그인들
	[
		'expo-build-properties',
		{
			android: {
				compileSdkVersion: 35,
				targetSdkVersion: 35,
				usesCleartextTraffic: true,
				extraMavenRepos: [
					'https://devrepo.kakao.com/nexus/content/groups/public/',
					'https://dl.buzzvil.com/public/maven',  // 버즈빌 저장소 추가
				],
			},
			ios: {
				useFrameworks: 'static',
			},
		},
	],
	// ... 기타 플러그인들
],
```

---

## 실제 사용하기

이제 만든 모듈을 실제로 사용해볼 시간이다!

### 모듈 import

```typescript
import buzzvilModule from '@buzzvil-module'
```

### 앱 초기화 시 SDK 초기화

**`App.js`**

```javascript
import buzzvilModule from '@buzzvil-module'

const initializeBuzzvil = useCallback(async () => {
	try {
		/** 버즈베네핏 초기화 */
		const BUZZVIL_APP_ID = isIos ? '000000000000000' : '000000000000000'

		await buzzvilModule.initialize(BUZZVIL_APP_ID)
	} catch (error) {
		console.warn('Buzzvil initialize error >>>>', error)
	}
}, [isIos])

useEffect(() => {
	initializeBuzzvil()
}, [])
```

### 사용자 로그인/로그아웃 처리

**`AuthContext.js`**

```javascript
// 로그인 시
const login = async ({ user, ... }) => {
	// ... 기존 로그인 로직

	/** 버즈베네핏 로그인 */
	try {
		// 로그인 상태 확인
		const isBuzzvilLogined = await buzzvilModule?.isLoggedIn()
		if (!isBuzzvilLogined) {
			// 유저 추가정보 가져오기
			const userAccountData = {
				gender: 'M',
				birth: '1991-01-04',
				user_token: 'user_token',
			}

			// 버즈베네핏 로그인 (모듈에서 요구하는 형태로 잘 전달해야한다!)
			await buzzvilModule.login(userAccountData?.user_token, {
				...(!!userAccountData?.gender && {
					gender: userAccountData?.gender?.toUpperCase(),
				}),
				...(!!userAccountData?.birth && {
					birthYear: Number(userAccountData?.birth?.slice(0, 4)),
				}),
			})
		}
	} catch (error) {
		console.warn('Buzzvil login error >>>>', error)
	}
}

// 로그아웃 시
const logout = async ({ ... }) => {
	// ... 기존 로그아웃 로직

	/** 버즈베네핏 로그아웃 */
	try {
		const isBuzzvilLogined = await buzzvilModule.isLoggedIn()
		if (isBuzzvilLogined) {
			await buzzvilModule.logout()
		}
	} catch (error) {
		console.warn('Buzzvil logout error >>>>', error)
	}
}
```

### 베네핏허브 화면 표시

```typescript
const showBuzzvilWaterMissionHandler = (): MissionHandler => ({
	handle: async () => {
		try {
			const isBuzzvilLogined = await buzzvilModule.isLoggedIn()

			// 로그인이 안되어있는 상태면 로그인 시켜주기
			if (!isBuzzvilLogined) {
				// 유저 추가정보 가져오기 (테스트 데이터)
				const userAccountData = {
					gender: 'M',
					birth: '1991-01-04',
					user_token: 'user_token',
				}

				// 버즈베네핏 로그인
				await buzzvilModule.login(userAccountData?.user_token, {
					...(!!userAccountData?.gender && {
						gender: userAccountData?.gender?.toUpperCase(),
					}),
					...(!!userAccountData?.birth && {
						birthYear: Number(userAccountData?.birth?.slice(0, 4)),
					}),
				})

				// 2초 후 다시 시도하기
				retryTimer.current = setTimeout(() => {
					showBuzzvilWaterMissionHandler()
				}, 2000)
				return
			}

			// 베네핏허브 화면 표시!
			await buzzvilModule.show()
		} catch (error) {
			console.warn('Buzzvil show error >>', error)
		}
	},
})
```

---

## 마무리!

네이티브 개발자이자 크로스플랫폼 개발자를 왔다갔다하면서 혼돈의 카오스였지만 포기하지않고 계속해서 공식문서를 뚫어져서 파해친 결과 결국 연동에 성공할 수 있었다!
이제는 RN SDK가 없더라도 웃으며 무엇이든지 모듈화 시킬 수 있는 자신감이 생겼습니다!
여러분도 해보세요! 허허허
---
title: Expo SDK 53 마이그레이션 삽질기 🔥
date: 2025-07-04 16:00:00 +0900
categories: [개발일지, React Native]
tags: [expo, react-native, migration, sdk53, 마이그레이션]
---

어느날 우리 앱에 새로운 라이브러리를 추가해야하는 날이 있었다.
그리고 작업 후 앱을 빌드해서 스토어에 배포하는 순간 일이 시작됬다.
iOS App Store에서 최소 빌드 버전 요구사항이 올라가는 바람에,
Expo 클라우드 빌드에서 제공해주는 iOS 빌드 버전을 올려야 했기에, 당연히 Expo SDK 53으로 업데이트를 진행하게 되었다.
단순한 버전 업데이트라고 생각했는데... 생각보다 많은 Breaking Changes가 있어서 힘들었다!

이번 마이그레이션 과정에서 마주친 여러 소소한(?) 이슈들과 해결방법들을 기록해두고 싶다.

---

## Expo SDK 53 업데이트

가장 기본적인 SDK 업데이트부터 시작했다.

```bash
yarn expo install expo@latest
npx expo install --fix
```

> **참고**: [Expo SDK 53 Changelog](https://expo.dev/changelog/sdk-53)

---

## @tanstack/react-query의 `remove()` & `isLoading` 이슈

React Query v5부터 `remove()` 메소드가 deprecated 되었다. 현재는 v4를 사용 중이지만 미래를 위해 미리 대응했다.

#### `remove()` 메소드 대체
```javascript
// Before
const { remove } = useQuery({})
remove()

// After
const queryClient = useQueryClient()
queryClient.removeQueries({ queryKey: ['key'] })
```

#### `isLoading` → `isPending` 변경
```javascript
// Before
const { mutate, isLoading } = useMutation({})

// After
const { mutate, isPending } = useMutation({})
```

---

## `queryClient.invalidateQueries()` 시그니처 변경

기존의 파라미터 전달 방식이 deprecated 될 예정이라 미리 수정했다.

```javascript
// Before
queryClient.invalidateQueries(queryKey)

// After
queryClient.invalidateQueries({ queryKey: queryKey })
```

---

## react-native-gesture-handler Touchable 컴포넌트 이슈

`react-native-gesture-handler`가 ~2.14.0에서 ~2.24.0으로 업데이트되면서 Touchable 시리즈가 deprecated 되었다.

#### Touchable 컴포넌트 단순화
기존의 `isOrigin` 옵션을 제거하고 `react-native`의 TouchableOpacity만 사용하도록 변경했다.

```javascript
// Before - 복잡한 조건부 렌더링, gesture-handler의 Touchable 시리즈 deprecated
{isOrigin && (
  <RNTouchableOpacity {...props}>
    {children}
  </RNTouchableOpacity>
)}
{!isOrigin && (
  <TouchableOpacity {...props}>
    {children}
  </TouchableOpacity>
)}

// After - 단순화, react-native의 TouchableOpacity 사용
<RNTouchableOpacity {...props}>
  {children}
</RNTouchableOpacity>
```

#### TouchableWithoutFeedback → Pressable 대체
```javascript
// Before
<TouchableWithoutFeedback onPress={Keyboard.dismiss} accessible={false}>
  <SearchAdRolling />
  <MyKeyword />
</TouchableWithoutFeedback>

// After
<Pressable onPress={Keyboard.dismiss}>
  <SearchAdRolling />
  <MyKeyword />
</Pressable>
```

---

## FlatList의 ListEmptyComponent Fragment 이슈

`react-native-gesture-handler`의 FlatList에서 ListEmptyComponent에 Fragment(`<></>`)를 최상위 컴포넌트로 사용할 수 없게 되었다.

```javascript
// Before
ListEmptyComponent={
  <>
    {!isLoading && isEnd && (
      ListEmptyComponent || <Empty text="목록내역이 없습니다." />
    )}
  </>
}

// After
ListEmptyComponent={
  <View>
    {!isLoading && isEnd && (
      ListEmptyComponent || <Empty text="목록내역이 없습니다." />
    )}
  </View>
}
```

---

## Image 컴포넌트 랜더링 중 상태 변화 에러

RN 버전 업데이트로 인해 이미지 컴포넌트 랜더링 중 사용되지 않는 state 변경으로 에러가 발생했다.

```javascript
// Before - 불필요한 state
const [imageLayout, setImageLayout] = useState({ width: 0, height: 0 })

const handleLayout = e => {
  if (typeof onLayout === 'function') {
    onLayout()
  }

  if (!error) return
  
  const { width, height } = e.nativeEvent.layout
  setImageLayout({ width, height }) // 사용되지 않는 state 변경
}

// After - state 제거
const handleLayout = e => {
  if (typeof onLayout === 'function') {
    onLayout()
  }
}
```

---

## @gorhom/bottom-sheet의 enableDynamicSizing 옵션

새로운 BottomSheet에서는 `enableDynamicSizing`을 false로 설정해야 기존 `snapPoints` 옵션을 사용할 수 있다.

```javascript
<BottomSheet
  ref={bottomSheetRef}
  snapPoints={snapPoints}
  onChange={handleChange}
  onClose={handleDismiss}
  enablePanDownToClose={enablePanDownToClose}
  enableDynamicSizing={false} // 추가 필수
  {...props}
/>
```

---

## use-latest-callback 라이브러리 충돌

Expo SDK 53 업데이트와 관련해 `use-latest-callback` 라이브러리에서 앱 크래시가 발생했다.

`package.json`에 안정적인 버전을 강제로 지정했다.

```json
{
  "resolutions": {
    "use-latest-callback": "0.1.9"
  }
}
```

---

## react-native-tab-view props 전달 이슈

RN 버전 업데이트에 따른 props 전달 방식 오류로 패치파일을 생성했다.

```diff
// patches/react-native-tab-view+3.5.2.patch
-      const props: TabBarItemProps<T> & { key: string } = {
-        key: route.key,
+      const props: TabBarItemProps<T> = {
         position: position,
         route: route,
         // ...
```

---

## expo-camera API 변경

Camera 관련 API가 대폭 변경되었다.

```javascript
// Before
import { Camera } from 'expo-camera'
const [permissionForCamera, requestPermissionForCamera] = Camera.useCameraPermissions()

<Camera type={type} className="flex-1" ref={ref => setCamera(ref)} />

// After
import { CameraView, useCameraPermissions } from 'expo-camera'
const [permission, requestPermission] = useCameraPermissions()

<CameraView className="flex-1" facing={type} ref={cameraRef} />
```

---

## 안드로이드 스플래시 이미지 사이즈 변경

Expo SDK 53부터 안드로이드 스플래시 이미지가 전체 화면에서 가운데 원형으로 변경되었다. 그래서 스플래시 이미지의 사이즈를 별도로 변경해줘야 했다.

```javascript
// app.config.js
export default {
  expo: {
    // ...
    ios: {
      splash: {
        image: './src/assets/splash_ios.png',
        resizeMode: 'contain',
        backgroundColor: '#ffffff',
      },
    },
    android: {
      splash: {
        image: './src/assets/splash_android.png', // 별도 이미지
        resizeMode: 'contain',
        backgroundColor: '#ffffff',
      },
    },
  },
}
```

---

## @react-native-firebase 인스턴스 방식 변경

Firebase 라이브러리가 앱 인스턴스를 생성하여 사용하는 방식으로 변경되었다.

### Analytics
```javascript
// Before
import analytics from '@react-native-firebase/analytics'
analytics().logEvent(eventName, eventParams)

// After
import { getAnalytics, logEvent } from '@react-native-firebase/analytics'
import { getApp } from '@react-native-firebase/app'

const firebaseApp = getApp()
const firebaseAnalytics = getAnalytics(firebaseApp)
logEvent(firebaseAnalytics, eventName, eventParams)
```

### Messaging
```javascript
// Before
import messaging from '@react-native-firebase/messaging'
const token = await messaging().getToken()

// After
import { getMessaging, getToken } from '@react-native-firebase/messaging'
const messaging = getMessaging(firebaseApp)
const token = await getToken(messaging)
```

---

## RefreshView ref 오버라이딩 이슈

props로 전달된 `ref: null`이 내부 ref를 오버라이딩하는 문제가 발생했다.

```javascript
// Before
<ScrollView
  ref={el => {
    scrollRef.current = el
    setProductScreenScroll(el)
  }}
  {...args} // ref: null이 포함된 경우 오버라이딩 발생
/>

// After
const { ref, ...rest } = args || {}

<ScrollView
  ref={el => {
    scrollRef.current = el
    setProductScreenScroll(el)
  }}
  {...(!!ref && { ref })} // ref가 있을 때만 전달
  {...rest}
/>
```

---

## react-native-webview decelerationRate 타입 이슈

WebView의 `decelerationRate`에서 문자열 값이 에러를 발생시켰다.

```javascript
// Before
<WebView
  decelerationRate={"normal"} // 문자열 에러
/>

// After
<WebView
  decelerationRate={isIos ? scrollDecelerationRate.ios : scrollDecelerationRate.android}
/>
```

---

## 마무리

Expo SDK 53 마이그레이션은... 쉽지않았다...
패치패키지를 이렇게 많이 한적은 없었으며, 의존하고있는 라이브러리들이 많을 수록 그만큼 리소스 비용도 증가한다는걸 느꼈다.
그리고 공식문서들을 꼭 반드시 잘 읽어보자!

---

> **관련 링크**  
> - [Expo SDK 53 Changelog](https://expo.dev/changelog/sdk-53)
> - [React Query v5 Migration Guide](https://tanstack.com/query/latest/docs/react/guides/migrating-to-v5)
---
title: 'useSyncExternalStore 이해하고 글로벌 상태 관리 시스템 사용해보기'
date: 2024-05-26 16:00:00 +0900
category: [개발일지, React]
tags: [React, useSyncExternalStore, 상태관리, 성능최적화, 글로벌상태, Tearing방지, 개발일지]
---

??? : 글로벌 상태 관리 어떤거 쓸까용?...

Redux는 뭔가 선뜻 호감이 가지 않네요...;;
Context는 Provider 지옥에 빠질 수 있는 부분이 염려가 됩니다...
이는 곧 앱 최적화에 치명적인 문제를 불러일으킬 수 있죠. 물론! 디버깅도 힘듭니다!
그래서 선택한 건 **useSyncExternalStore**를 활용한 가볍고 저렴한! 커스텀 글로벌 상태 관리 훅을 만들기! 입니다.

(외부 라이브러리에 의존하는건 더욱 지양하는터라...)
"그냥 최상위에서 동작하는 상태 관리를 만들자~" 가 시작이였습니다!

---

## useSyncExternalStore 이해하기

React 18에서 새로 나온 이 훅은 **외부 스토어와 React를 안전하게 연결**해줍니다!


### 예제

```javascript
const value = useSyncExternalStore(subscribe, getSnapshot)
```

- **subscribe**: 스토어 변경을 구독하는 함수
- **getSnapshot**: 현재 스토어 값을 가져오는 함수


### 왜 이게 좋은가?

1. **Tearing 방지**

   - 유저의 동시성 렌더링에서 발생할 수 있는 상태 불일치 문제 해결 가능
   - 모든 컴포넌트가 항상 일관된 상태를 보게 됨

2. **성능 최적화** (getter와 setter를 분리 사용 목표)

   - 필요한 컴포넌트만 리렌더링 설계 가능
   - 구독/해지가 자동으로 관리됨

---

## 우리만의 스토어 시스템 구축하기

### 기본 스토어 만들기

먼저 가장 기본이 되는 스토어부터 만들어봤어요.
utils.js 파일에 스토어 생성 함수를 만들었습니다!

```javascript
// utils.js
eexport function createState(initialState) {
	let state = initialState.default

  // Set을 사용해서 리스너 중복 방지
	const listeners = new Set()

	return {
		// 현재 상태 가져오기
		get: () => state,

		// 상태 업데이트하기
		set: () => {
			return newState => {
				// 함수형 업데이트 지원
				const nextState = typeof newState === 'function' ? newState(state) : newState

				// 상태가 실제로 바뀐 경우에만 동작
				if (nextState !== state) {
					state = nextState
					listeners.forEach(listener => listener(state))
				}
			}
		},

		// 구독 관리
		subscribe: listener => {
			listeners.add(listener)

			// 구독 해제 함수 반환
			return () => listeners.delete(listener)
		},
	}
}
```

여기서 **Set**을 사용한 이유는...

- 중복 구독 방지 (같은 함수를 여러 번 등록해도 한 번만 실행)
- 빠른 추가/삭제 (시간 복잡도)

### 각각 사용할 스토어 상태 정의하기

자기만의 스토어 상태를 정의하도록 app.ts 라는 파일에 작성했습니다!
key는 스토어 상태를 구분하기 위해 사용합니다.
default는 스토어 상태의 초기값을 설정합니다.

```javascript
// app.ts
export const userState = createState({
	key: 'app/userState',
	default: {},
})

export const alertState = createState({
	key: 'app/alertState',
	default: { visible: null },
})

export const routeNameState = createState({
	key: 'app/routeNameState',
	default: '',
})
```

---

## [getter & setter, getter, setter] 쓰임새에 따라 사용할 수 있는 3종 훅 만들기

### useStoreState (읽기 쓰기 모두 가능)

```javascript
export default function useStoreState(store, selector) {
	try {
		// 데이터 어뎁팅을 위해 selector 사용 가능
		let getSnapShot = () => store.get()
		if (typeof selector === 'function') {
			getSnapShot = () => selector(store.get())
		}

		// useSyncExternalStore 사용
		const state = useSyncExternalStore(store.subscribe, getSnapShot)

		return [state, store.set()]
	} catch (error) {
		console.warn('error in useStoreState', error)
	}
}
```

### useStoreStateValue (읽기 전용)

```javascript
// 상태만 필요하고 set 함수는 필요 없을 때
export function useStoreStateValue(store, selector) {

	let getSnapShot = () => store.get()
	if (typeof selector === 'function') {
		getSnapShot = () => selector(store.get())
	}

	// set 함수 없이 상태만 반환
	const state = useSyncExternalStore(store.subscribe, getSnapShot)
	return state
}
```

### useSetStoreState (쓰기 전용)

```javascript
// set 함수만 필요하고 상태는 필요 없을 때 (리렌더링 방지)
export function useSetStoreState(store) {
	validateStore(store)

	// useSyncExternalStore를 사용하지 않음 = 리렌더링 없음
	return store.set()
}
```

---

## 사용 예시

#### 읽기 + 쓰기가 모두 필요한 컴포넌트

```javascript
function UserProfile() {
	const [user, setUser] = useStoreState(userState)

	const handleUpdateName = newName => {
		setUser({ ...user, name: newName })
	}

	return (
		<div>
			<h1>안녕하세요, {user.name}님!</h1>
			<button onClick={() => handleUpdateName('새이름')}>이름 변경</button>
		</div>
	)
}
```

#### 읽기만 필요한 컴포넌트

```javascript
function UserGreeting() {
	// set 함수가 없어서 더 가벼움
	const user = useStoreStateValue(userState)

	return <span>안녕하세요, {user.name}님!</span>
}
```

#### 쓰기만 필요한 컴포넌트 (리렌더링 방지)

```javascript
function LogoutButton() {
	// 상태 변경해도 이 컴포넌트는 리렌더링 안됨!
	const setUser = useSetStoreState(userState)

	const handleLogout = () => {
		setUser({})
		// 다른 컴포넌트들은 리렌더링되지만 이 컴포넌트는 안됨
	}

	return <button onClick={handleLogout}>로그아웃</button>
}
```

---

## Selector를 활용해서 특정 데이터만 포커싱 하기

너무나 큰 데이터에서 특정 부분만 포커싱 하고 싶을 때 selector 함수를 전달해서 사용할 수 있도록 설계 했습니다!

```javascript
const bigState = createState({
	default: {
		user: { name: 'John', age: 30 },
		ui: { theme: 'dark', language: 'ko' },
		data: { posts: [], comments: [] },
	},
})

function UserNameDisplay() {
	// user.name만 바뀔 때만 리렌더링!
	const userName = useStoreStateValue(bigState, state => state.user.name)

	return <h1>{userName}</h1>
}

function ThemeToggle() {
	// ui.theme만 바뀔 때만 리렌더링!
	const theme = useStoreStateValue(bigState, state => state.ui.theme)
	const setState = useSetStoreState(bigState)

	const toggleTheme = () => {
		setState(prev => ({
			...prev,
			ui: { ...prev.ui, theme: theme === 'dark' ? 'light' : 'dark' },
		}))
	}

	return <button onClick={toggleTheme}>현재 테마: {theme}</button>
}
```

---

## 마무리

글로벌 상태 관리 시스템을 활용하면서...

- **단순함의 힘**: 복잡한 도구보다 필요에 맞는 단순한 도구가 더 효과적일 수 있다
- **개발자 경험**: 도구는 개발자가 쓰기 편해야 한다

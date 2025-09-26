---
title: 'useSyncExternalStore 이해하고 글로벌 상태 관리 시스템 사용해보기'
date: 2024-05-26 16:00:00 +0900
category: [개발일지, React]
tags: [React, useSyncExternalStore, 상태관리, 성능최적화, 글로벌상태, Tearing방지, 개발일지]
---

# 글로벌 상태 관리가 필요한데 어떻게 해야할까?

Redux는 뭔가 호감이 가지 않고...
Context는 불필요한 리렌더링이 많고...
그래서 우리 팀이 선택한 건 **useSyncExternalStore**를 활용한 커스텀 글로벌 상태 관리 시스템!

## 왜 또 다른 상태 관리 도구를 만들었나?

### 기존 도구들의 아쉬운 점들

**Redux**

- 너무 무겁고 보일러플레이트가 많음
- 작은 상태 하나 바꾸려고 액션, 리듀서 다 만들어야 함
- 간단한 전역 상태 하나만 쓰고 싶은데...

**Context** 🟠

- Provider 지옥 (Provider 안에 Provider 안에 Provider...)
- 하나의 값이 바뀌면 모든 하위 컴포넌트가 리렌더링
- 성능 최적화하면서 느끼는점은 역시... "이게 진짜 최적화 맞지? 🫠"

**Zustand, Jotai 등** 🟡

- 외부 라이브러리에 의존하지는 않기로 했어요...

그냥 최상위에서 동작하는 상태 관리를 만들자~

## 🔍 useSyncExternalStore가 뭐길래?

React 18에서 새로 나온 이 훅은 **외부 스토어와 React를 안전하게 연결**해주는 역할을 해요.

### 핵심 개념

```javascript
const value = useSyncExternalStore(subscribe, getSnapshot)
```

- **subscribe**: 스토어 변경을 구독하는 함수
- **getSnapshot**: 현재 스토어 값을 가져오는 함수

### 왜 이게 좋은가?

1. **Tearing 방지** 🛡️

   - 유저의 동시성 렌더링에서 발생할 수 있는 상태 불일치 문제 해결 가능
   - 모든 컴포넌트가 항상 일관된 상태를 보게 됨

2. **성능 최적화** ⚡

   - 필요한 컴포넌트만 리렌더링 설계 가능
   - 구독/해지가 자동으로 관리됨


## 우리만의 스토어 시스템 구축하기

### 기본 스토어 만들기

먼저 가장 기본이 되는 스토어부터 만들어봤어요.

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

## 생성한 스토어 사용하기

### useStoreState (읽기 쓰기 모두 가기)

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

## 🏗 실제 사용 예시

### 1. 상태 정의하기

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

### 2. 컴포넌트에서 사용하기

**읽기 + 쓰기가 모두 필요한 컴포넌트**

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

**읽기만 필요한 컴포넌트**

```javascript
function UserGreeting() {
	// set 함수가 없어서 더 가벼움
	const user = useStoreStateValue(userState)

	return <span>안녕하세요, {user.name}님!</span>
}
```

**쓰기만 필요한 컴포넌트 (리렌더링 방지)**

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

## 🎯 Selector를 활용한 세밀한 구독

큰 객체에서 특정 부분만 구독하고 싶을 때:

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

## 🔚 마무리

글로벌 상태 관리 시스템을 활용하면서...

- **단순함의 힘**: 복잡한 도구보다 필요에 맞는 단순한 도구가 더 효과적일 수 있다
- **개발자 경험**: 도구는 개발자가 쓰기 편해야 한다

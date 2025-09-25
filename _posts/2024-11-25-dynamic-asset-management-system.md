---
title: '앱 경량화의 딜레마 해결기 - 동적 에셋 관리 시스템 구축하기'
date: 2024-11-25 16:00:00 +0900
categories: [개발일지, React Native]
tags: [React Native, 게임개발, 앱최적화, 에셋관리, AsyncStorage, 경량화, 서버최적화]
---

# 게임 앱 경량화의 딜레마 해결기 ⚖️

모바일 게임 개발에서 **앱 용량**과 **서버 비용** 사이의 트레이드오프는 항상 고민거리다. 우리 팀도 농장 게임을 운영하면서 이 딜레마에 직면했고, 창의적인 해결책을 찾아 구현한 경험을 공유하려고 한다.

## 📋 문제 상황

### 초기 문제점들

1. **앱 용량 증가** 📈

   - 게임 특성상 수많은 이미지 에셋 필요
   - 작물, UI, 아이템 등 다양한 그래픽 리소스
   - 앱스토어 다운로드 허들 증가

2. **하드코딩된 게임 설정** 🔧

   - 게임 밸런스 값들이 코드에 직접 입력
   - 작은 수정에도 전체 앱 업데이트 필요
   - CodePush 배포 빈도 증가로 인한 부담

3. **업데이트 프로세스 복잡성** 🔄
   - PM/PD의 게임 밸런스 조정 요청
   - 개발자의 코드 수정 → 빌드 → 배포 과정
   - 빠른 대응의 어려움

## 💡 해결 아이디어

**"서버에서 동적으로 에셋과 설정을 관리하되, 불필요한 중복 호출은 방지하자!"**

### 핵심 아이디어

- 🌐 **서버 중앙화**: 이미지와 게임 설정을 서버에서 관리
- 📱 **로컬 캐싱**: 한 번 받은 데이터는 로컬에 저장
- 🔄 **버전 기반 업데이트**: 변경사항이 있을 때만 새로 다운로드
- ⚡ **효율적 동기화**: 헬스체크 방식으로 최소한의 네트워크 사용

## 🛠 시스템 아키텍처

### 1. 데이터 구조 설계

```javascript
// 서버에서 제공하는 Static Data 구조
const staticDataStructure = {
  crop_assets: {
    // 작물 관련 이미지 URL들
    tomato: "https://cdn.example.com/crops/tomato.png",
    carrot: "https://cdn.example.com/crops/carrot.png"
  },
  game_assets: {
    // UI 관련 이미지 URL들
    buttons: {...},
    icons: {...}
  },
  data: {
    // 게임 설정값들
    crop_growth_time: 3600, // 1시간
    max_inventory: 100,
    invite_reward: {...}
  },
  ver: 2401011200 // 버전 타임스탬프
}
```

### 2. 버전 관리 시스템

가장 핵심적인 부분인 버전 체크 로직:

```javascript
const updateFarmStaticData = async () => {
	try {
		// 1. 서버에서 현재 최신 버전 확인
		const { ver: serverVersion } = await checkStaticVersion()

		// 2. 로컬에 저장된 데이터와 버전 확인
		const localData = JSON.parse(await AsyncStorage.getItem(FARM_STATIC_DATA))
		const localVersion = localData?.ver || 0

		// 3. 버전 비교로 업데이트 필요 여부 판단
		const needsUpdate = !localData || localVersion < serverVersion

		if (needsUpdate) {
			// 4. 새 데이터 다운로드
			const newData = await getStaticData()

			// 5. 로컬 스토리지에 저장
			await AsyncStorage.setItem(
				FARM_STATIC_DATA,
				JSON.stringify({
					...newData,
					ver: serverVersion,
				})
			)
		}

		return true
	} catch (error) {
		console.error('Static data update failed:', error)
		return false
	}
}
```

### 3. 효율적인 Hook 설계

React Hook으로 깔끔하게 추상화:

```javascript
export const useFarmStaticData = () => {
	const [farmStaticData, setFarmStaticData] = useState(null)

	// 로컬 데이터 로드
	const loadLocalData = async () => {
		const data = JSON.parse(await AsyncStorage.getItem(FARM_STATIC_DATA))
		setFarmStaticData(data)
	}

	// 컴포넌트 마운트 시 로컬 데이터 로드
	useEffect(() => {
		loadLocalData()
	}, [])

	return {
		farmStaticData, // 현재 사용 가능한 데이터
		updateFarmStaticData, // 업데이트 함수
	}
}
```

## 🔧 구현 세부사항

### 1. 버전 체크 API

서버에서는 두 개의 경량화된 엔드포인트 제공:

```javascript
// 버전만 확인하는 경량 API
const checkStaticVersion = async () => {
	const response = await fetch('/api/farm/static-version')
	return response.json() // { ver: 2401011200 }
}

// 실제 데이터를 받아오는 API
const getStaticData = async () => {
	const response = await fetch('/api/farm/static-data')
	return response.json() // 전체 에셋과 설정 데이터
}
```

### 2. 에러 처리 및 폴백

통신에 실패하더라도 게임은 진행할 수 있도록 하는 정책을 반영하기위해 에러 로깅만 하고 게임 진행을 위한 처리는 하지 않는다.

```javascript
const updateFarmStaticData = async () => {
	try {
		// ... 업데이트 로직
		return true
	} catch (error) {
		// 통계 전송으로 에러 모니터링
		clientErrorToStatistics({
			key: 'update-farm-static-data-error',
			error: String(error),
		})

		// 에러 발생 시 게임 입장 차단
		return false
	}
}

// 사용하는 곳에서
const canEnterGame = await updateFarmStaticData()
if (!canEnterGame) {
	// 에러 메시지 표시 및 재시도 유도
	showErrorMessage('게임 데이터를 불러오는데 실패했어요')
}
```

### 3. 특별한 데이터 처리

일부 데이터는 추가 가공이 필요했다.

```javascript
// 이벤트 보상 데이터 파싱
const parsedInviteReward = await parseInviteEventReward(
	needsUpdate ? newData.invite_reward : localData.invite_reward.origin
)

// 최종 저장 데이터 구성
const finalData = {
	...rawData,
	data: {
		...rawData.data,
		invite_reward: parsedInviteReward, // 가공된 데이터
	},
	ver: serverVersion,
}
```

## 📊 성과 및 효과

### 1. 앱 용량 최적화 📱

- **Before**: 모든 에셋 포함 → 100MB+ 앱 크기
- **After**: 필수 에셋만 포함 → 50MB 이하로 경량화
- **결과**: 다운로드 허들 대폭 감소

### 2. 서버 비용 절약 💰

- **Before**: 매 실행마다 에셋 요청 → 수만 명 × 매일 호출
- **After**: 버전 체크만 → 업데이트 시에만 실제 다운로드
- **결과**: 네트워크 트래픽 50% 이상 감소

### 3. 개발 프로세스 개선 🚀

- **Before**: 설정 변경 → 코드 수정 → 전체 배포
- **After**: 서버 설정 변경 → 즉시 반영
- **결과**: PM/PD 요청 대응 시간 단축

### 4. 사용자 경험 향상 ✨

- 앱 다운로드 속도 향상
- 게임 실행 시 빠른 로딩
- 설정 변경 시 즉시 반영

## 🎯 핵심 설계 원칙

### 1. **레이지 로드 방식을 통해 필요한 시점에만 데이터 로필**
계
```javascript
// 필요한 시점에만 데이터 로드 (업데이트 타임 체크 필요)
useEffect(() => {
	if (shouldEnterFarm) {
		updateFarmStaticData()
	}
}, [shouldEnterFarm])
```

### 2. **최대한 로컬 데이터를 사용하도록 설계**

```javascript
// 로컬 데이터 우선 사용
const loadData = async () => {
	// 1. 로컬 데이터 먼저 로드
	const localData = await getLocalData()
	setData(localData)

	// 2. 백그라운드에서 업데이트 체크
	checkForUpdates()
}
```

## 🚨 잠깐! 신규 사용자는 로컬에 데이터가 없잖아?

**문제**: 신규 사용자는 로컬에 데이터가 없음

**해결책**:

```javascript
const initializeStaticData = async () => {
	const localData = await getLocalData()

	if (!localData) {
		// 신규 사용자는 무조건 다운로드
		showLoadingScreen('게임 데이터 준비 중...')
		await forceUpdateStaticData()
	}
}
```


## 🔚 마무리

이 프로젝트를 통해 **"완벽한 해결책은 없지만, 창의적인 접근으로 여러 문제를 동시에 해결할 수 있다"**는 것을 배웠다.

### 핵심 교훈들:

- 🎯 **사용자 중심 사고**: 다운로드 경험과 앱 성능 모두 고려
- 💰 **비용 효율성**: 개발 비용과 운영 비용의 균형

이제 PM이 "이 값 좀 바꿔주세요"라고 하면 "네, 5분 후에 반영됩니다"라고 답할 수 있게 되었다! 🎉

---
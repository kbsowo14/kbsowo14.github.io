---
title: '게임 서비스가 있는 앱 경량화 하기! - 에셋 관리 시스템'
date: 2024-11-25 16:00:00 +0900
categories: [개발일지, React Native]
tags: [React Native, 게임개발, 앱최적화, 에셋관리, AsyncStorage, 경량화, 서버최적화]
---

서버 비용은 너무나 비싸다!
당연히 요즘 서버 호출이 조금만 늘어도 연락이온다! (서비스에 무슨일 있냐며...)

서비스에 무슨일이 있긴합니다! 새로운 이벤트 및 서비스로 인해 내부 객체를 갱신하거나 리패치를 돌려야하는 동작이 점점 생기게 됬으니!

아무리 생각해도 반드시 리패치는 돌려야한다. 실시간의 데이터 동기화가 유저마다 필요했기 때문이다.
그렇다면 방법은 하나다. 리패치 항목중 불필요한 리패치를 최소화 해보자!
그래서 결국 고민한건 특별한 업데이트가 아니면 앱 리소스로 관리하는게 좋겠다!

게임 내에서 사용하는 설정 값들과 에셋들을 앱 스토리지에 저장을 시켜놓고 사용하기로 했다!

---

## 문제점들...

1. **앱 용량 증가**
   - 게임 특성상 수많은 이미지 에셋 필요
   - 작물, UI, 아이템 등 다양한 그래픽 리소스

2. **하드코딩된 게임 설정**
   - 게임 밸런스 값들이 코드에 직접 입력
   - 작은 수정에도 전체 앱 업데이트 필요
   - CodePush 배포 빈도 증가로 인한 부담

3. **업데이트 프로세스 복잡성**
   - PM/PD의 게임 밸런스 조정 요청
   - 개발자의 코드 수정 → 빌드 → 배포 과정
   - 빠른 대응의 어려움

---

## 아이디어...

**"서버에서 동적으로 에셋과 설정을 관리하되, 불필요한 중복 호출은 방지하자!"**

- **서버 중앙화**: 이미지와 게임 설정을 서버에서 관리
- **로컬 캐싱**: 한 번 받은 데이터는 로컬에 저장
- **버전 기반 업데이트**: 변경사항이 있을 때만 새로 다운로드
- **효율적 동기화**: 헬스체크 방식으로 최소한의 네트워크 사용

---

## 구현 방법

### 1. 데이터 형태 미리보기 (사실 아래 예시 데이터보다 훠얼~씬 많고 크다)

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

### 2. 버전 체크 로직 구현하기

가장 핵심적인 부분인 버전 체크 로직!
데이터를 불러오는 API를 호출하기전에
checkStaticVersion API를 호출하여 현재 버전과 서버에 등록되어있는 버전을 비교해서 업데이트 필요 여부를 파악한다!
(물론 초기에는 없으니 바로 호출하겠지만)

버전의 형태는 날짜를 연결한 Number 형태로 관리한다. 20241025 < 20241026

```javascript
// 버전 체크 API
const checkStaticVersion = async () => {
	const response = await fetch('/api/farm/static-version')
	return response.json() // { ver: 20241026 }
}

// 버전 체크 및 데이터 업데이트
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

### 3. 저장되어있는 데이터를 사용하게 해주는 Hook 구현하기

이 Hook은 저장되어있는 데이터를 사용하게 해주는 Hook이다.
물론 상단에 구현한 업데이트 함수도 제공해준다.

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

## 사용 예시



### 1. 앱 입장 및 포커싱이 될때마다 업데이트 로직 실행

```javascript
const canEnterGame = await updateFarmStaticData()
if (!canEnterGame) {
	// 에러 메시지 표시 및 재시도 유도
	showErrorMessage('게임 데이터를 불러오는데 실패했어요')
}
```

### 2. 저장된 데이터 사용하기

```javascript
const { farmStaticData } = useFarmStaticData()
console.log(farmStaticData)
```
---

## 효과들...

### 1. 앱 용량 최적화
- **Before**: 모든 에셋 포함 → 100MB+ 앱 크기
- **After**: 필수 에셋만 포함 → 50MB 이하로 경량화

### 2. 서버 비용 절약 (트래픽 50% 감소)
- **Before**: 매 실행마다 에셋 요청 → 수만 명 × 매일 호출
- **After**: 버전 체크만 → 업데이트 시에만 실제 다운로드

### 3. 개발 프로세스 개선 (이제 프론트의 코드 푸시 없이 바로 적용 가능)
- **Before**: 설정 변경 → 코드 수정 → 전체 배포
- **After**: 서버 설정 변경 → 즉시 반영

---

## 마무리

이 프로젝트를 통해 **"완벽한 해결책은 없지만, 창의적인 접근으로 여러 문제를 동시에 해결할 수 있다"**는 것을 배웠다.
이제 PM이 "이 값 좀 바꿔주세요"라고 하면 "네, 5분 후에 반영됩니다"라고 답할 수 있게 되었다!

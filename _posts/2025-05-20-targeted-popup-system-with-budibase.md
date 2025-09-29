---
title: 유저 타겟팅 전면 팝업 시스템 구축하기 - 부디베이스 활용
date: 2025-05-20 16:00:00 +0900
category: [개발일지, React Native]
tags: [React Native, 부디베이스, 타겟팅, 전면팝업, 마케팅시스템, 사용자경험, 개발일지]
---

게임을 입장 할 때마다 홍보의 목적이나 가이드 목적으로 반드시 사용되는 기능은 역시 전면 팝업 기능이다.
유저가 게임에 포커싱이 될때마다 매번 나타나는 기능이기 때문에, 그만큼 매번 프론트의 업데이트를 통해 변화를 줄 수 없는 것이다.
또한 유저들에게 무분별한 전면팝업을 보여주는 것은 유저 경험에 너무 좋지 않다!

마케터 분들과 기획자 분들이 개발자와의 커뮤니케이션 비용 없이 직접 특정 유저들을 선택하거나 그룹핑하고 특정 요일, 특정 시간대를 지정하고, 해당 팝업을 터치하면 원하는 화면으로 이동하거나 원하는 모달을 노출 시킬 수 있도록 구현하는 것이 중요했다.
그래서 부디베이스를 붙여 자유롭게 세팅하고 제거할 수 있도록 구현했다.

## 핵심 컨셉

- **정교한 타겟팅**: 시간, 요일, 레벨, 노출횟수 등 다차원 조건
- **노코드 관리**: 부디베이스로 마케터가 직접 관리
- **데이터 기반**: 실시간 조건 검증 및 노출 제어
- **즉시 반영**: 코드 배포 없이 실시간 적용

## 부디베이스에서 넘어오는 데이터 형태는 어떻게 하면 좋을까?

```javascript
// 부디베이스 팝업 데이터 예시
const popupData = {
	theme: 'christmas_event', // 팝업 식별자
	main_image: [
		{
			url: 'https://cdn.example.com/popup.jpg',
			name: '크리스마스 이벤트',
		},
	],
	info: {
		level: { value: 5, type: 'over' }, // 5레벨 초과인 유저
		water_pot: { value: 10, type: 'under' }, // 물의 양 10g 미만인 유저
		nutrient: { value: 50, type: 'equal' }, // 영양분 정확히 50개인 유저
	},
	repeat_days: ['Mon', 'Tue', 'Wed'], // 월,화,수만 노출
	repeat_start_time: '0900', // 오전 9시부터
	repeat_end_time: '1800', // 오후 6시까지
	impressions: 3, // 하루 최대 3회 노출
	button_landing_type: 'route', // 클릭 시 이동 타입
	button_landing: 'EventScreen', // 이동할 화면
	start_at: '2025-12-20', // 시작일
	end_at: '2025-12-31', // 종료일
}
```

## 타겟팅 조건 검증 로직은 어떻게 구현해야할까?

가장 핵심인 타겟팅 조건 검증 로직을 구현했다.
유저의 농장 작물 데이터를 기반으로한 데이터 검증이 우선이고,
요일과 시간대 조건도 검증하도록 했다. (사실 이 시간대는 반복 시간대를 의미하고 이미 visible 되는 시간대는 엔드포인트 쪽에서 필터링이 된다.)

```javascript
const validateTargetingConditions = (popupData, userFarmData, currentTime) => {
	const { info = {}, repeat_days, repeat_start_time, repeat_end_time } = popupData
	const { level, water_pot, normal_fertilizer, good_fertilizer, nutrient } = userFarmData

	let conditionChecks = []

	// 1. 농장 상태 조건 검증
	Object.keys(info).forEach(key => {
		const condition = info[key]
		const userValue = userFarmData[key]

		switch (condition.type) {
			case 'equal':
				conditionChecks.push(userValue === condition.value ? 1 : 0)
				break
			case 'under':
				conditionChecks.push(userValue < condition.value ? 1 : 0)
				break
			case 'over':
				conditionChecks.push(userValue > condition.value ? 1 : 0)
				break
		}
	})

	// 2. 요일 조건 검증
	if (repeat_days?.length > 0) {
		const todayIndex = dayjs().day()
		const dayNames = ['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat']
		const todayName = dayNames[todayIndex]
		const isValidDay = repeat_days.includes(todayName)
		conditionChecks.push(isValidDay ? 1 : 0)
	}

	// 3. 시간대 조건 검증
	if (repeat_start_time && repeat_end_time) {
		const currentTime = dayjs().format('HHmm')
		const isValidTime =
			Number(currentTime) >= Number(repeat_start_time) &&
			Number(currentTime) <= Number(repeat_end_time)
		conditionChecks.push(isValidTime ? 1 : 0)
	}

	// 모든 조건이 만족되어야 함
	return conditionChecks.every(check => check === 1)
}
```

## 매일 노출되면 그것 또한 피로하다!

유저마자 노출되는 횟수도 제한해서 적당히 선을 넘지않게(?) 노출해보자.
로컬 스토리지에 해당 팝업의 고유한 테마 String 값과 impressions 수를 저장해놓고,
노출이 될때마다 하나씩 증가시키고, 최대 노출 횟수를 초과하면 노출하지 않는다.

```javascript
const manageImpressionLimit = async (popupList, maxImpressions) => {
	// 로컬 스토리지에서 노출 기록 조회
	const impressionHistory = JSON.parse(await AsyncStorage.getItem(FARM_POPUP_SHOW_INFO)) || []

	const today = dayjs().format('YYYY-MM-DD')

	// 오늘 노출 가능한 팝업들 필터링
	const availablePopups = popupList.filter(popup => {
		const history = impressionHistory.find(h => h.theme === popup.theme)

		// 기록이 없거나, 하루가 지났거나, 노출 횟수가 제한보다 적은 경우
		if (!history) return true

		const isDifferentDay = dayjs().diff(history.date, 'day') >= 1
		if (isDifferentDay) return true

		return history.impressions < popup.impressions
	})

	return availablePopups
}
```

## React Hook으로 구현해서 어디서든 전면팝업을 쉽게 노출할 수 있도록 구현해보자.

```javascript
const useFarmFrontPopup = () => {
	const [popupList, setPopupList] = useState([])
	const [visiblePopupList, setVisiblePopupList] = useState(null)

	// 부디베이스에서 팝업 데이터 조회
	const fetchPopupData = useCallback(async () => {
		const rawData = await farmApi.farmManager.getFarmPopup[1]({ // budibase API 호출
			env: getAppEnv(),
			visible: true,
			is_equal_search: true,
			ongoing_start_col_name: 'start_at',
			ongoing_end_col_name: 'end_at',
		})

		// 현재 농장 상태 조회
		const farmData = await farmApi.farm.getMy[1]({})
		const { user_crop, warehouse } = farmData

		const currentUserData = {
			level: user_crop?.level,
			water_pot: warehouse?.water_pot,
			n_fertilizer: warehouse?.n_fertilizer,
			g_fertilizer: warehouse?.g_fertilizer,
			nutrient: user_crop?.nutrient,
		}

		// 조건에 맞는 팝업들만 필터링
		const targetedPopups = rawData.filter(popup =>
			validateTargetingConditions(popup, currentUserData, dayjs())
		)

		setVisiblePopupList(targetedPopups)
	}, [])

	// 팝업 노출 실행
	const showFarmFrontPopup = useCallback(async () => {
		try {
			const availablePopups = await manageImpressionLimit(visiblePopupList)
			if (!availablePopups.length) return

			const targetPopup = availablePopups[0]

			// 노출 기록 업데이트
			await updateImpressionHistory(targetPopup.theme)

			// 통계 전송
			sendShowStatistics(targetPopup)

			// 팝업 표시
			setFarmFrontPopup(targetPopup)
		} catch (error) {
			console.error('팝업 노출 오류:', error)
		}
	}, [visiblePopupList])

	return {
		data: popupList,
		visibleList: visiblePopupList,
		show: showFarmFrontPopup,
	}
}
```

## 마케터 친화적으로 구현하기위해서는 직관적으로 설정할 수 있게 해야한다!

부디베이스 인터페이스에서 마케터가 쉽게 설정

```javascript
// 조건 타입별 설정 예시
const conditionTypes = {
	equal: '정확히 일치', // level = 5
	under: '미만', // water_pot < 10
	over: '초과', // nutrient > 100
}

// 시간 설정 예시
const timeSettings = {
	repeat_days: ['Mon', 'Wed', 'Fri'], // 월,수,금만
	repeat_start_time: '1000', // 오전 10시부터
	repeat_end_time: '1500', // 오후 3시까지
}

// 등등...
```

아래는 부디베이스의 데이터 형태다.
![budi-01](/assets/img/budi-01.png)
![budi-02](/assets/img/budi-02.png)


## 마무리

이 프로젝트를 통해 **"기술은 사용자를 위한 것"**이라는 것을 다시 한번 깨달았다. 마케터도 사용자이고, 게임 유저도 사용자다. 두 사용자 모두를 만족시키는 시스템을 만드는 것이 진정한 성공이었다!

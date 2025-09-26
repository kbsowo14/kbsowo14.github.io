---
title: '내가 보려고 만든 브릿지 페이지 생성 가이드'
date: 2023-10-21 16:00:00 +0900
category: [개발일지, React]
tags: [React, Next.js, 브릿지페이지, 딥링크, 앱스토어, 개발일지]
---

# 브릿지 페이지(Bridge Page) 구축 가이드

## 🎯 브릿지 페이지란?

브릿지 페이지는 외부 링크를 통해 유입된 사용자를 모바일 앱으로 연결해주는 중간 페이지입니다.
사용자의 디바이스를 감지하여 앱이 설치되어 있으면 앱을 실행하고, 없으면 앱스토어로 안내하는 역할을 합니다. (MMP 솔루션 사용)

**핵심 기능:**
- 📱 모바일에서는 앱 실행 → 앱스토어 이동
- 💻 PC에서는 웹사이트로 이동
- 🔗 소셜 공유 시 예쁜 미리보기 제공
- 📊 사용자 유입 추적 및 분석

## 🏗️ 전체 프로젝트 구조

```
bridge-page/
├── app/                    # Next.js App Router
│   ├── bridge/            # 브릿지 메인 페이지
│   │   └── page.tsx       # 동적 메타데이터 + 리다이렉션
│   └── layout.tsx         # 루트 레이아웃
├── components/            # React 컴포넌트
│   └── BridgeComponent.tsx # 브릿지 로직 처리
├── public/               # 정적 파일
│   └── bridge.html       # Airbridge SDK + 앱 실행 로직
├── libs/                 # 유틸리티
│   └── api.ts            # API 통신
└── next.config.js        # Next.js 설정
```

## 🚀 기술 스택

- **Next.js 13+** (App Router)
- **TypeScript**
- **Airbridge SDK** (MMP 솔루션)
- **TailwindCSS** (스타일링)

## 📝 단계별 구현 가이드

### Step 1: 프로젝트 초기 설정

```bash
# Next.js 프로젝트 생성
npx create-next-app@latest bridge-page --typescript --tailwind --app
cd bridge-page

# 필수 패키지 설치
npm install axios
```

### Step 2: 핵심 파일들 생성하기

#### 🔧 `public/bridge.html` - 앱 실행의 핵심!

이 파일이 실제로 앱을 실행하는 역할을 합니다.

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>앱으로 이동 중...</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
</head>
<body>
  <div id="loading">앱을 실행하는 중입니다...</div>
  
  <!-- Airbridge SDK -->
  <script>
    (function (a_, i_, r_, _b, _r, _i, _d, _g, _e) {
      a_[_b] = a_[_b] || {};
      a_[_b].queue = a_[_b].queue || [];
      a_[_b].loaded = 0;
      a_[_b].init = function (a, b, c) {
        var d = i_.createElement(r_);
        d.async = 1;
        d.src = "https://static.airbridge.io/sdk/latest/airbridge.min.js";
        d.onload = function () {
          a_[_b].loaded = 1;
          a_[_b].init(a, b, c);
        };
        i_.head.appendChild(d);
      };
    })(window, document, "script", "airbridge");

    // SDK 초기화 및 앱 실행 로직
    window.addEventListener("load", function() {
      // Airbridge 초기화
      airbridge.init({
        app: "your-app-name",
        appToken: "your-app-token"
      });

      // URL 파라미터 가져오기
      const urlParams = new URLSearchParams(window.location.search);
      const targetUrl = urlParams.get('url') || 'home';
      const deepLink = `yourapp://${targetUrl}`;

      // 디바이스 감지 및 앱 실행
      const isIOS = /iphone|ipad|ipod/i.test(navigator.userAgent);
      const isAndroid = /android/i.test(navigator.userAgent);
      
      if (isIOS || isAndroid) {
        // 모바일: 앱 실행 시도
        airbridge.setDeeplinks({
          deeplinks: {
            ios: deepLink,
            android: deepLink
          },
          fallbacks: {
            ios: "https://apps.apple.com/app/your-app-id",
            android: "https://play.google.com/store/apps/details?id=com.yourapp"
          },
          buttonID: "openButton",
          redirect: true
        });
      } else {
        // PC: 웹사이트로 이동
        window.location.href = "https://your-website.com";
      }
    });
  </script>

  <!-- 자동 실행 버튼 (숨김) -->
  <button id="openButton" style="display: none;">앱 열기</button>
  
  <script>
    // 페이지 로드 후 자동으로 앱 실행 (화면을 몇초는 보여줘야하니까)
    setTimeout(() => {
      document.getElementById('openButton').click();
    }, 3000);
  </script>
</body>
</html>
```

#### `app/bridge/page.tsx` - 메타데이터 + 리다이렉션

```typescript
import { Metadata } from 'next';
import BridgeComponent from '@/components/BridgeComponent';

interface Props {
  searchParams: {
    uid?: string;
    url?: string;
    title?: string;
    description?: string;
    image?: string;
  };
}

// 동적 메타데이터 생성 (generateMetadata 함수 사용)
export async function generateMetadata({ searchParams }: Props): Promise<Metadata> {
  const { uid, title, description, image } = searchParams;
  
  // uid가 있으면 API에서 메타데이터 조회, 없으면 URL 파라미터 사용
  let metadata = {
    title: title || '앱으로 이동',
    description: description || '앱에서 더 나은 경험을 즐기세요',
    image: image || '/default-og-image.jpg'
  };

  if (uid) {
    try {
      const response = await fetch(`${process.env.NEXT_PUBLIC_API_BASE_URL}/api/metadata?uid=${uid}`);
      const data = await response.json();
      metadata = data;
    } catch (error) {
      console.error('메타데이터 조회 실패:', error);
    }
  }

  return {
    title: metadata.title,
    description: metadata.description,
    openGraph: {
      title: metadata.title,
      description: metadata.description,
      images: [metadata.image],
    },
    twitter: {
      card: 'summary_large_image',
      title: metadata.title,
      description: metadata.description,
      images: [metadata.image],
    }
  };
}

export default function BridgePage({ searchParams }: Props) {
  return <BridgeComponent searchParams={searchParams} />;
}
```

#### `components/BridgeComponent.tsx` - 리다이렉션 로직

```typescript
'use client';

import { useEffect } from 'react';
import { useRouter } from 'next/navigation';

interface Props {
  searchParams: {
    uid?: string;
    url?: string;
    [key: string]: string | undefined;
  };
}

export default function BridgeComponent({ searchParams }: Props) {
  const router = useRouter();

  useEffect(() => {
    // URL 파라미터를 bridge.html로 전달
    const params = new URLSearchParams();
    
    Object.entries(searchParams).forEach(([key, value]) => {
      if (value) params.append(key, value);
    });

    // bridge.html로 리다이렉션 (실제 앱 실행)
    const redirectUrl = `/bridge.html?${params.toString()}`;
    router.replace(redirectUrl);
  }, [searchParams, router]);

  // 리다이렉션 중 보여줄 로딩 화면
  return (
    <div className="min-h-screen flex items-center justify-center bg-gradient-to-br from-blue-500 to-purple-600">
      <div className="text-center text-white">
        <div className="animate-spin rounded-full h-12 w-12 border-b-2 border-white mx-auto mb-4"></div>
        <h1 className="text-xl font-semibold">앱으로 이동하는 중...</h1>
        <p className="text-blue-100 mt-2">잠시만 기다려주세요</p>
      </div>
    </div>
  );
}
```


### Step 3: 환경 설정 파일들

#### `next.config.js` - Next.js 설정

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  // 메인 페이지를 브릿지로 리다이렉션
  async redirects() {
    return [
      {
        source: '/',
        destination: '/bridge',
        permanent: false,
      },
    ];
  },
  
  // 이미지 최적화 설정
  images: {
    domains: ['your-cdn.com', 'your-cms.com'],
  },
};

module.exports = nextConfig;
```

#### 🔐 `.env.local` - 환경 변수

```bash
# Airbridge 설정
NEXT_PUBLIC_AIRBRIDGE_APP_NAME=your-app-name
NEXT_PUBLIC_AIRBRIDGE_APP_TOKEN=your-app-token

# 앱 정보
NEXT_PUBLIC_APP_SCHEME=yourapp
NEXT_PUBLIC_IOS_APP_ID=your-ios-app-id
NEXT_PUBLIC_ANDROID_PACKAGE_ID=com.yourapp.package

# API 설정
NEXT_PUBLIC_API_BASE_URL=https://your-api.com

# 앱스토어 URL
NEXT_PUBLIC_IOS_STORE_URL=https://apps.apple.com/app/your-app-id
NEXT_PUBLIC_ANDROID_STORE_URL=https://play.google.com/store/apps/details?id=com.yourapp
```

## 🚀 사용 방법

### 기본 사용법
```
https://your-bridge.com/bridge?url=game/level/1
```

### 메타데이터와 함께 사용
```
https://your-bridge.com/bridge?uid=content123&url=game/level/1
```

### URL 파라미터 직접 전달
```
https://your-bridge.com/bridge?url=game/level/1&title=레벨1 도전&description=새로운 레벨에 도전해보세요&image=https://cdn.com/level1.jpg
```

## 📱 사용자 플로우

```
1. 사용자가 공유된 링크 클릭
   ↓
2. /bridge 페이지 로드 (메타데이터 표시)
   ↓
3. BridgeComponent가 /bridge.html로 리다이렉션
   ↓
4. bridge.html에서 디바이스 감지 (실제 앱 실행 로직 담당)
   ↓
5-a. 모바일: 앱 실행 시도 → 실패 시 앱스토어
5-b. PC: 웹사이트로 이동
```

---
title: Jekyll과 GitHub Pages로 블로그 만들기
date: 2024-09-25 15:00:00 +0900
categories: [개발, Jekyll]
tags: [jekyll, github-pages, 블로그, 웹개발]
image:
  path: /assets/img/posts/jekyll-logo.png
  alt: Jekyll 로고
---

## 🚀 Jekyll 블로그 구축기

최근에 Jekyll과 GitHub Pages를 이용해서 개인 블로그를 만들었습니다. 
그 과정에서 배운 것들을 정리해보려고 합니다.

### ✨ Jekyll이란?

Jekyll은 정적 사이트 생성기(Static Site Generator)로, 마크다운 파일을 HTML로 변환해주는 도구입니다.

**장점:**
- 🎯 **빠른 로딩**: 정적 파일이라 속도가 빠름
- 💰 **무료 호스팅**: GitHub Pages에서 무료로 호스팅 가능
- 📝 **마크다운 지원**: 글 작성이 간편함
- 🎨 **테마 지원**: 다양한 테마 선택 가능

### 🛠️ 설치 및 설정

#### 1. Ruby 설치
```bash
# macOS의 경우
brew install ruby

# 버전 확인
ruby -v
```

#### 2. Jekyll 설치
```bash
gem install jekyll bundler
```

#### 3. 새 블로그 생성
```bash
jekyll new my-blog
cd my-blog
bundle exec jekyll serve
```

### 🎨 Chirpy 테마 적용

이 블로그는 [Chirpy 테마](https://github.com/cotes2020/jekyll-theme-chirpy)를 사용했습니다.

**Chirpy 테마의 특징:**
- 📱 반응형 디자인
- 🌙 다크/라이트 모드 지원
- 🔍 검색 기능
- 📊 카테고리/태그 시스템
- 💬 댓글 시스템 지원

### 📝 글 작성 방법

Jekyll에서 글을 쓸 때는 다음 규칙을 따라야 합니다:

#### 파일명 규칙
```
_posts/YYYY-MM-DD-제목.md
```

#### Front Matter 예시
```yaml
---
title: 글 제목
date: YYYY-MM-DD HH:MM:SS +0900
categories: [카테고리1, 카테고리2]
tags: [태그1, 태그2, 태그3]
pin: true  # 상단 고정
---
```

### 🚀 GitHub Pages 배포

1. GitHub 저장소 생성 (`username.github.io`)
2. 코드 푸시
3. Settings > Pages에서 배포 설정
4. 몇 분 후 `https://username.github.io`에서 확인

### 💡 팁과 트릭

#### 이미지 최적화
```markdown
![이미지 설명](/assets/img/posts/image.jpg)
_이미지 캡션_
```

#### 코드 블록
```javascript
function hello() {
    console.log("Hello, World!");
}
```

#### 수식 지원
수학 공식도 지원합니다: $E = mc^2$

### 🎯 앞으로의 계획

- [ ] 댓글 시스템 설정 (Giscus)
- [ ] Google Analytics 연동
- [ ] SEO 최적화
- [ ] 더 많은 컨텐츠 작성!

---

Jekyll로 블로그를 만드는 것은 생각보다 어렵지 않았습니다. 
정적 사이트의 장점을 활용하면서도 충분히 기능적인 블로그를 만들 수 있어서 만족스럽습니다!

다음 포스트에서는 더 자세한 커스터마이징 방법을 다뤄보겠습니다. 🚀

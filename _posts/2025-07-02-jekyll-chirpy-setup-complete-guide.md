---
title: "Jekyll Chirpy 테마로 GitHub Pages 블로그 만들기 - 시행착오 완전 정복"
date: 2025-07-02 20:00:00 +0900
categories: [Blog, Jekyll]
tags: [jekyll, chirpy, github-pages, tutorial]
description: "Windows 11에서 Jekyll Chirpy 테마로 GitHub Pages 블로그를 만들면서 겪은 모든 시행착오와 해결 방법을 상세히 기록합니다."
image:
  path: /assets/img/posts/chirpy-setup-guide.jpg
  alt: Jekyll Chirpy Setup Guide
---

> 이 글은 Claude (Anthropic)의 도움을 받아 작성되었습니다. 실제 블로그 구축 과정에서 발생한 모든 오류와 해결 과정을 담았습니다.

## 서론

Jekyll Chirpy 테마로 GitHub Pages 블로그를 만들면서 예상치 못한 수많은 오류를 만났습니다. 공식 문서만으로는 해결하기 어려운 문제들이 많았고, 특히 Windows 11 환경에서의 설정은 더욱 까다로웠습니다. 이 글은 제가 겪은 모든 시행착오와 해결 방법을 상세히 기록한 것입니다.

## 환경 설정

- **OS**: Windows 11
- **IDE**: Visual Studio Code
- **Terminal**: PowerShell
- **Ruby**: 3.3
- **Node.js**: LTS 버전

## 1단계: Chirpy 테마 Fork하기

### Chirpy Starter vs Full Theme

처음에는 간단해 보이는 [chirpy-starter](https://github.com/cotes2020/chirpy-starter)를 사용하려 했지만, 이는 gem 기반 테마로 `_sass`, `_includes` 등의 핵심 파일을 수정할 수 없었습니다. 

완전한 커스터마이징을 원한다면 **전체 Chirpy 테마**를 fork해야 합니다:

1. [jekyll-theme-chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) 접속
2. Fork 버튼 클릭
3. Repository name을 `username.github.io` 형식으로 변경

## 2단계: GitHub 잔디 문제 해결

Fork한 저장소는 기본적으로 GitHub contribution(잔디)이 기록되지 않습니다. 이를 해결하기 위해:

```powershell
# Fork한 저장소 클론
git clone https://github.com/YourUsername/yourusername.github.io.git
cd yourusername.github.io

# 기존 git 히스토리 삭제
Remove-Item -Recurse -Force .git

# 새로운 git 저장소로 초기화
git init
git add .
git commit -m "Initial commit: Setup Jekyll blog with full Chirpy theme"

# GitHub 저장소 연결
git remote add origin https://github.com/YourUsername/yourusername.github.io.git
git branch -M main
git push -u origin main --force
```

## 3단계: 시행착오 #1 - Ruby 버전 충돌

### 문제
```
can't find gem bundler (= 2.6.7) with executable bundle (Gem::GemNotFoundException)
```

### 해결
```powershell
# Gemfile.lock 삭제
Remove-Item Gemfile.lock -Force -ErrorAction SilentlyContinue

# 최신 bundler 설치
gem install bundler

# 의존성 재설치
bundle install
```

## 4단계: 시행착오 #2 - Husky Commit 규칙

### 문제
```
✖   subject may not be empty [subject-empty]
✖   type may not be empty [type-empty]
husky - commit-msg script failed
```

Chirpy는 개발자용 커밋 규칙을 강제합니다.

### 해결
개인 블로그에는 불필요하므로 제거:

```powershell
# Husky 및 관련 파일 제거
Remove-Item .husky -Recurse -Force
Remove-Item .commitlintrc.json -Force -ErrorAction SilentlyContinue
Remove-Item commitlint.config.js -Force -ErrorAction SilentlyContinue

# 이제 자유롭게 커밋 가능
git commit -m "원하는 커밋 메시지"
```

## 5단계: GitHub Actions 설정

Chirpy의 기본 워크플로우는 개발용입니다. GitHub Pages 배포용 워크플로우가 필요합니다:

```powershell
# starter 폴더의 배포 템플릿 활성화
Copy-Item .github\workflows\starter\pages-deploy.yml .github\workflows\pages-deploy.yml

# 개발용 워크플로우 정리 (선택사항)
Remove-Item .github\workflows\ci.yml -Force
Remove-Item .github\workflows\cd.yml -Force
# ... 기타 개발용 파일들
```

## 6단계: 시행착오 #3 - SCSS 빌드 오류

### 문제
```
Error: Can't find stylesheet to import.
@use 'vendors/bootstrap';
```

### 해결
Bootstrap 등의 vendor 파일이 누락되어 발생. Node.js 의존성을 설치하고 빌드해야 합니다:

```powershell
# Node.js 의존성 설치
npm install

# Assets 빌드
npm run build

# Bootstrap SCSS 복사
New-Item -ItemType Directory -Force -Path _sass\vendors
Copy-Item -Path node_modules\bootstrap\scss -Destination _sass\vendors\bootstrap -Recurse -Force
```

## 7단계: 시행착오 #4 - npm ci 오류

### 문제
GitHub Actions에서:
```
The `npm ci` command can only install with an existing package-lock.json
```

### 해결
```powershell
# package-lock.json 생성
npm install

# 커밋
git add package-lock.json
git commit -m "fix: add package-lock.json for npm ci"
git push origin main
```

## 8단계: 시행착오 #5 - 환경 보호 규칙

### 문제
```
Branch "main" is not allowed to deploy to github-pages due to environment protection rules
```

### 해결
1. GitHub 저장소 → Settings → Environments
2. `github-pages` 환경 클릭
3. Protection rules에서:
   - Required reviewers: 체크 해제
   - Deployment branches: `All branches` 선택

## 9단계: 최종 GitHub Actions 워크플로우

모든 문제를 해결한 완전한 워크플로우:

```yaml
name: "Build and Deploy"

on:
  push:
    branches:
      - main
    paths-ignore:
      - .gitignore
      - README.md
      - LICENSE
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          submodules: true

      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v4

      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.3'
          bundler-cache: true

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Build Assets
        run: |
          npm ci
          npm run build

      - name: Build site
        run: bundle exec jekyll b -d "_site"
        env:
          JEKYLL_ENV: "production"

      - name: Upload site artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: "_site"

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

## 정리

### 핵심 체크리스트

- [ ] 전체 Chirpy 테마 fork (starter 아님)
- [ ] Git 히스토리 초기화 (잔디 기록용)
- [ ] Husky 제거 (자유로운 커밋)
- [ ] GitHub Pages 워크플로우 설정
- [ ] Node.js 의존성 및 vendor 파일 추가
- [ ] package-lock.json 커밋
- [ ] 환경 보호 규칙 해제

### 작업 흐름

이후 포스트 작성 시:

```powershell
# 새 포스트 작성
$date = Get-Date -Format "yyyy-MM-dd"
$title = "post-title"
New-Item -ItemType File -Path "_posts\$date-$title.md"

# 로컬 테스트
bundle exec jekyll serve

# 배포
git add .
git commit -m "post: 새 글 제목"
git push origin main
```

## 마치며

Jekyll과 Chirpy 테마는 강력하지만 초기 설정이 복잡합니다. 특히 Windows 환경에서는 예상치 못한 문제들이 많이 발생합니다.

이 글이 같은 문제를 겪고 있는 분들께 도움이 되길 바랍니다.

---

*이 글은 Claude (Anthropic)의 도움을 받아 작성되었으며, 실제 블로그 구축 과정에서 발생한 모든 오류와 해결 방법을 담고 있습니다.*
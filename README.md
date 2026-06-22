# 관상 운세 (Face Fortune)

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Vue](https://img.shields.io/badge/Vue-3.x-brightgreen.svg)
![Vite](https://img.shields.io/badge/Vite-6.x-purple.svg)

## 프로젝트 소개

관상 운세는 인공지능 기반의 얼굴 분석 애플리케이션으로, 사용자의 얼굴 특징을 분석하여 흥미로운 성격·직업·결혼 운세 분석 결과를 생성합니다. 본 프로젝트는 오락용이며, 얼굴 인식 기술과 전통 관상학을 결합해 새로운 인터랙티브 경험을 만들어냅니다.

## 주요 기능

- **실시간 얼굴 검출**: 카메라로 사용자의 얼굴을 실시간으로 포착
- **얼굴 특징 분석**: 얼굴 윤곽, 표정 등의 특징을 분석
- **성격 특성 평가**: 레이더 차트로 성격 특성 분석을 직관적으로 표시
- **운세 예측**: 직업, 결혼 등 다양한 방면의 운세 분석 포함
- **오늘의 길흉**: 개인화된 일상 조언 제공
- **반응형 디자인**: 다양한 화면 크기의 기기에 대응

## 기술 스택

- **프런트엔드 프레임워크**: Vue 3 + Vite
- **UI 컴포넌트 라이브러리**: Element Plus
- **차트 시각화**: ECharts
- **얼굴 인식**: face-api.js
- **개발 언어**: JavaScript

## 프로젝트 스크린샷

*(스크린샷 추가 예정)*

## 설치 및 사용

### 환경 요구사항

- Node.js 16.0+
- npm 또는 yarn

### 설치 단계

1. 프로젝트를 로컬에 클론

```bash
git clone https://github.com/your-username/face-fortune.git
cd face-fortune
```

2. 의존성 설치

```bash
npm install
# 또는
yarn
```

3. 개발 서버 실행

```bash
npm run dev
# 또는
yarn dev
```

4. 프로덕션 빌드

```bash
npm run build
# 또는
yarn build
```

## 프로젝트 구조

```
face-fortune/
├── public/               # 정적 리소스
│   └── models/           # face-api.js 모델 파일
├── src/                  # 소스 코드
│   ├── assets/           # 프로젝트 리소스 파일
│   ├── components/       # 컴포넌트
│   │   └── FaceFortune.vue  # 주요 기능 컴포넌트
│   ├── App.vue           # 루트 컴포넌트
│   ├── main.js           # 진입 파일
│   └── style.css         # 전역 스타일
├── index.html            # HTML 템플릿
├── vite.config.js        # Vite 설정
└── package.json          # 프로젝트 의존성
```

## 사용 방법

1. 「카메라 열기」 버튼을 클릭해 카메라 접근 권한을 허용합니다
2. 얼굴이 카메라 화각 안에 들어오고 조명이 충분한지 확인합니다
3. 「운세 보기 시작」 버튼을 클릭해 관상 분석을 진행합니다
4. 성격 특성, 직업 분석, 결혼 분석 등 분석 결과를 확인합니다
5. 오늘의 길흉 조언을 확인할 수 있습니다

## 유의사항

- 본 애플리케이션은 카메라 권한이 필요하므로 브라우저에서 권한이 부여되었는지 확인하세요
- 얼굴 인식 결과는 조명, 각도 등의 요인에 영향을 받습니다
- 분석 결과는 오락용이므로 진지하게 받아들이지 마세요

## 향후 계획

- [ ] 더 많은 관상 분석 항목 추가
- [ ] 분석 기록 저장 지원
- [ ] 더 많은 얼굴 특징 인식 모델 추가
- [ ] 모바일 환경 경험 개선
- [ ] 다국어 지원

## 기여 가이드

코드 기여, 이슈 제기, 제안을 환영합니다! 다음 단계를 따라주세요:

1. 본 저장소를 Fork 합니다
2. 기능 브랜치를 생성합니다 (`git checkout -b feature/amazing-feature`)
3. 변경 사항을 커밋합니다 (`git commit -m 'Add some amazing feature'`)
4. 브랜치에 푸시합니다 (`git push origin feature/amazing-feature`)
5. Pull Request를 엽니다

## 라이선스

본 프로젝트는 MIT 라이선스를 따릅니다 - 자세한 내용은 [LICENSE](LICENSE) 파일을 참고하세요

## 연락처

문의나 제안이 있으면 GitHub Issues를 통해 연락해 주세요.

---

**면책 조항**: 본 애플리케이션은 오락용일 뿐이며, 분석 결과는 어떠한 과학적 근거도 없으므로 중요한 의사결정에 사용하지 마세요.

# Three.js 기반 디지털 트윈 웹 애플리케이션 분석 보고서

본 문서는 `ThreeJS_korean_bus.html`과 `smart_classroom_camera_stable.html` 두 파일의 실행 방법과 전체적인 코드 구조를 분석한 문서입니다. 두 파일 모두 외부 3D 모델(GLTF/OBJ) 파일 없이 순수 Three.js 기본 도형과 코드로만 3D 환경을 렌더링하는 **프로시저럴(Procedural) 모델링** 방식을 채택한 단일 파일 웹 애플리케이션입니다.

---

## 1. 실행 방법 (How to Run)

두 파일은 HTML, CSS, JavaScript가 하나의 파일에 통합된 형태이며, Three.js 라이브러리를 외부 CDN을 통해 불러옵니다. ES6 모듈(`type="module"`)을 사용하므로 보안 정책(CORS) 오류를 방지하기 위해 로컬 웹 서버 환경에서 실행하는 것이 권장됩니다.

### 방법 A: VS Code 확장을 이용한 실행 (가장 추천)
1. Visual Studio Code 편집기에서 분석할 `.html` 파일을 엽니다.
2. 확장 프로그램(Extensions) 탭에서 **Live Server** (제작자: Ritwick Dey)를 설치합니다.
3. 편집기 우측 하단의 **Go Live** 버튼을 클릭하거나, HTML 코드 창에서 우클릭 후 **Open with Live Server**를 선택합니다.
4. 기본 브라우저가 열리며 `http://127.0.0.1:5500/...` 주소에서 3D 화면이 실행됩니다.

### 방법 B: Python 로컬 서버 이용 (개발자용)
1. 터미널(또는 명령 프롬프트)을 열고 해당 HTML 파일이 있는 폴더로 이동합니다.
2. 다음 명령어를 입력하여 로컬 서버를 실행합니다.
   * Windows/Mac: `python -m http.server 8000`
3. 웹 브라우저를 열고 `http://localhost:8000`에 접속하여 HTML 파일을 클릭합니다.

*(참고: 외부 텍스처 이미지나 3D 모델 에셋을 불러오지 않고 코드 내부에서 Canvas 텍스처를 생성하여 사용하므로, 일부 최신 브라우저에서는 파일을 더블 클릭하여 바로 열어도 작동할 수 있습니다. 단, 로컬 서버 실행이 가장 안정적입니다.)*

---

## 2. 전체적인 문법 및 구조 분석

두 파일은 웹 프론트엔드 표준 기술인 HTML5, CSS3, Vanilla JavaScript(ES6+)를 사용하여 작성되었으며, 구조적으로 다음과 같은 4개의 주요 블록으로 나뉩니다.

### 2.1. HTML 구조 (UI 레이아웃)
`<div id="app">` 또는 `<div id="stage">`와 같은 컨테이너를 바탕으로 사용자 인터페이스(UI)를 구성합니다.
* **디지털 트윈 버스 (`ThreeJS_korean_bus.html`)**: Grid 레이아웃을 사용해 좌측 제어 패널(공정 단계), 중앙 3D 뷰포트, 우측 모니터링 패널로 구성되어 있습니다.
* **스마트 교실 (`smart_classroom_camera_stable.html`)**: 화면 전체를 3D 뷰어(stage)로 덮고, 그 위에 `position: absolute`를 이용해 플로팅 형태의 반투명 유리 질감(Glassmorphism) 제어 패널(슬라이더, 차트, 버튼 등)을 띄운 형태입니다. 교실 파일은 `<script type="application/json" id="sceneSpec">`를 통해 공간의 설정값(메타데이터)을 분리해 둔 특징이 있습니다.

### 2.2. CSS 스타일링 (`<style>`)
* **변수 사용**: `:root`에 `--bg`, `--teal`, `--gold` 등 CSS 변수를 선언하여 테마 색상을 일관되게 관리합니다.
* **반응형 웹**: `@media (max-width: ...)` 미디어 쿼리를 적극 활용하여 모바일 기기에서도 UI가 깨지지 않고 하단 메뉴나 햄버거 메뉴로 전환되도록 설계되었습니다.
* **모던 UI 기법**: `backdrop-filter: blur(...)`를 적용하여 3D 화면 위에 겹치는 패널들이 세련된 반투명 유리 질감을 띠도록 구성했습니다.

### 2.3. 모듈 임포트 (`<script type="importmap">`)
Node.js 환경(Webpack/Vite 등) 없이 브라우저에서 직접 ES 모듈을 로드하기 위해 **Import Map** 기능을 사용합니다.
```html
<script type="importmap">
  {
    "imports": {
      "three": "https://cdn.jsdelivr.net/npm/three@0.180.0/build/three.module.js",
      "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.180.0/examples/jsm/"
    }
  }
</script>
```
이후 `<script type="module">` 블록 안에서 `import * as THREE from 'three';` 구문을 통해 라이브러리를 깔끔하게 불러와 사용합니다.

### 2.4. JavaScript 메인 로직 (`<script type="module">`)
Vanilla JS로 DOM을 제어하고, Three.js API를 활용해 3D 공간을 구축하는 핵심부입니다. 두 파일의 공통적인 로직 흐름은 다음과 같습니다.

1. **초기 셋업 (Initialization)**:
   * `THREE.Scene()`, `THREE.PerspectiveCamera()`, `THREE.WebGLRenderer()` 생성 및 초기화.
   * `OrbitControls`를 연결하여 마우스 드래그로 화면 회전/확대/축소 구현.
   * `HemisphereLight`, `DirectionalLight` 등을 추가하여 그림자(Shadow)와 조명 환경 설정.

2. **프로시저럴 3D 모델링 함수 (Procedural Modeling)**:
   * 외부 3D 툴(Blender 등) 없이 코드로 형태를 만들기 위해 래퍼(Wrapper) 함수를 자체 정의하여 사용합니다.
   * 예: `bx(부모, 너비, 높이, 깊이, 재질, x, y, z)` ➔ `BoxGeometry` 생성 후 위치 지정 및 부모 그룹에 추가.
   * 이 방식을 반복하여 뼈대, 바퀴, 벽면, 책상, 사람 등을 레고 조립하듯 구성합니다.

3. **상태 관리 및 UI 연동 (State & UI Sync)**:
   * HTML의 `<button>`, `<input type="range">` 요소에 `addEventListener`를 걸어 사용자 입력을 받습니다.
   * **버스 공장**: `setStage()` 함수를 통해 진행 단계에 따라 `THREE.Vector3.lerp`를 사용해 부품이 날아와 조립되는 애니메이션을 연산합니다. `Raycaster`를 이용해 3D 화면 속 부품을 클릭하는 기능도 포함되어 있습니다.
   * **스마트 교실**: 슬라이더 값을 읽어 조도, CO2 농도, 실내 온도 등 상태 객체(`sim`)를 업데이트하고, 이 데이터에 따라 3D 환경의 빛 강도(`intensity`)나 창문 블라인드 위치 등을 동적으로 제어합니다. 또한 Canvas API를 활용해 실시간 차트를 그립니다.

4. **렌더링 루프 (Animation Loop)**:
   * `renderer.setAnimationLoop()` 또는 `requestAnimationFrame()` 안에서 매 프레임마다 화면을 다시 그립니다(`renderer.render()`).
   * 이 루프 안에서 로봇 팔의 회전, 카메라의 부드러운 이동(Tweening), 공조기 바람 효과, 부품 조립 이동 위치 등을 지속적으로 계산(수학적 보간법 Lerp 활용)합니다.

## 요약
이 파일들은 프론트엔드 빌드 툴 없이 단일 HTML 파일만으로 복잡한 3D 디지털 트윈을 훌륭하게 구현한 코드 모범 사례(Boilerplate)입니다. 3D 에셋을 외부에서 불러오지 않고 코드로 모두 작성했기 때문에, 파일 용량이 매우 작고 Three.js의 핵심 기하학 및 재질 조작 원리를 이해하기에 아주 좋은 구조를 가지고 있습니다.
# 네이처 아쿠아리움 — 실감 수영 모션 & 물고기 디자인

이 프로젝트는 **브라우저만으로 실행 가능한 3D 디지털 수족관**입니다. Three.js와 커스텀 셰이더로 수면 잔광(가우스틱), 식생의 흔들림, 군영(boids) 기반 유영, **실제 물고기 같은 몸통 S-곡선 undulation + 꼬리/지느러미 파동**을 구현했습니다. 오브젝트를 클릭하면 **오늘의 명언**이 HUD로 나타납니다.

---

## 데모 실행 (Zero‑build)

* `index.html`(캔버스의 v4 파일)을 아무 정적 서버에서 열면 됩니다.

  * 예) **VS Code Live Server**, `npx serve`, `python -m http.server` 등.
* 로컬 파일을 더블클릭해도 동작하지만, 일부 환경에서 import map이 제한될 수 있어 **HTTP 서버 사용 권장**.

> 의존성은 CDN(UNPKG)의 ES Modules를 import map으로 매핑했습니다. 별도 설치가 필요 없습니다.

---

## 기술 스택

* **Three.js 0.161.0** (ESM, import map)
* **OrbitControls** (카메라 뷰 탐색)
* **Custom GLSL Shaders**

  * 바닥: 단순 가우스틱 느낌의 노이즈 셰이더
  * 식생: InstancedMesh + 버텍스 흔들림
  * 물고기: `onBeforeCompile`로 MeshStandardMaterial 버텍스 변형(몸통 undulation), 꼬리/지느러미는 반투명 파형 셰이더

---

## 주요 기능

* **군영 유영(Boids + 경로 추종)**: 회전 방향(yaw) 기반의 **완만한 bank(롤) 적용**으로 빙글빙글 돌던 문제 제거.
* **실감 모션**

  * 속도에 비례해 **몸통 굴곡 진폭/주파수**와 **꼬리/지느러미 파동**이 변화.
  * **Pitch/Pan/Roll 제한**으로 자연스러운 헤딩 유지.
* **식생 & 하드스케이프**

  * 수초 카펫/덤불/줄기를 InstancedMesh로 수천 개 렌더.
  * 바위/유목은 절차적 메시 + PBR.
* **인터랙션**: 씬의 오브젝트를 클릭하면 **카테고리별 명언** 토스트가 떠오름.
* **런타임 테스트**: 필수 요소를 `console.assert`로 점검(아래 “테스트” 참조).

---

## 파일 구성

본 프로젝트는 **단일 HTML** 안에 모든 코드가 포함되어 있습니다.

* `type="importmap"`으로 CDN 경로 매핑
* `type="module"`에서 로직 실행

---

## 구성/튜닝 포인트 (코드 상수)

```js
// 개체 수, 속도 범위, 힘 제한, 최소 속도, 롤 제한
const FISH_COUNTS = { cardinal: 30, rummy: 20, angelfish: 2, goldfish: 1 };
const SPEEDS      = { school:[3.2,4.8], drifter:[1.5,2.5] };
const MAXFORCE    = { school: 6, drifter: 5 };
const MIN_SPEED   = 0.9;
const BANK_LIMIT  = 0.6; // rad
```

* **색상 팔레트**: `basePalette`, `bellyPalette`
* **경로(ROUTE)**: `CatmullRomCurve3` 제어점을 바꿔 수조 내 **순환 동선/선회 반경** 변경
* **식생 밀도**: `carpetCount`, `bushCount`, `stemCount`

---

## 아키텍처 개요

* **씬/카메라/렌더러** 초기화 → 조명(헤미 + 방향광 2) → 바닥(가우스틱 셰이더) → 바위/유목 → 식생(InstancedMesh 3종) → 물고기 스폰(4종) → 업데이트 루프(`tick`)에서

  1. 셰이더 유니폼 시간 업데이트
  2. 보이드 스티어(응집·정렬·분리 + 경로 추종 + 부력)
  3. 헤딩/뱅크 보정, **몸통 undulation/지느러미 파동** 업데이트
  4. HUD/테스트 어설션

---

## 상호작용

* **클릭**: 레이캐스트로 카테고리를 찾고 `quoteBox`에 명언 출력(3초 페이드아웃)
* **카메라**: OrbitControls(팬 비활성), 줌 제한(18–48)

---

## 테스트 (런타임 어설션)

코드에 다음 **검증 로직**이 들어있습니다.

* 물고기 수가 설정과 일치
* 유목 ≥ 3, 카펫 ≥ 3000, 줄기 ≥ 1000
* 바닥은 **불투명**(레이어링 이슈 방지)
* 식생이 바닥보다 **항상 나중에 렌더**(renderOrder)
* 적어도 하나 이상의 물고기에 **꼬리 파트 존재**
* 물고기 몸통 재질에 **swim uniforms 주입** 확인
* 각 물고기에 **뱅크 제한값** 존재
* HUD 문자열이 **숫자/형식 정상**, `NaN/undefined` 미포함

### 추가 테스트 예시 (원하면 붙여넣어 사용)

```js
// 평균 속도가 범위 내인지 체크
const avgV = fishes.reduce((s,f)=>s+f.userData.vel.length(),0)/fishes.length;
console.assert(avgV>=MIN_SPEED && avgV<=6, `avgV out of range: ${avgV}`);

// 피치/롤 제한 준수
console.assert(fishes.every(f=>{
  const e = new THREE.Euler().setFromQuaternion(f.quaternion,'YXZ');
  return Math.abs(e.x)<=0.36 && Math.abs(e.z)<=BANK_LIMIT+1e-3;
}), 'pitch/roll limits violated');
```

---

## 트러블슈팅 (이번 작업에서 해결한 대표 이슈)

1. **`three` 모듈 해석 실패**: import map 누락/경로 문제 → `type="importmap"` 추가 및 CDN 매핑으로 해결.
2. **`mouse.x` undefined**: 포인터 NDC 좌표 미초기화 → `Vector2`를 전역으로 두고 이벤트에서 갱신.
3. **GLB 403**: 외부 호스트 권한 문제 → **절차적 메시** 기반으로 변경(의존성 제거). 필요 시 동일 도메인/허용된 CDN으로 교체.
4. **`instanceMatrix` 재선언 셰이더 에러**: InstancedMesh는 엔진이 속성을 주입 → 셰이더에서 `attribute mat4 instanceMatrix;` **선언 금지**.
5. **템플릿 문자열 중첩 에러**: `... + ${bushCount}` 형태 제거 → 단일 표현식으로 정리.

---

## 성능 팁

* 해상도: `renderer.setPixelRatio(Math.min(devicePixelRatio, 2))`
* 인스턴스 수 조절: `carpetCount / bushCount / stemCount`
* 지오메트리 세그먼트 축소(식생/바위)
* 모바일: 카메라 거리 증가, 톤매핑 노출 낮추기, 애니메이션 강도 축소

---

## 확장 아이디어

* **종별 파라미터 분리**: 엔젤/금붕어/카디널/러미노즈 각각의 undulation 주파수·진폭, 꼬리 파동, 보이드 반경 튜닝
* **코랄(테이블·버섯형)**: InstancedMesh + 노멀 기반 그라디언트 셰이더로 추가 가능
* **GLTF/SkinnedMesh 도입**: 셰이더 대신 실제 리깅 애니메이션 사용(동일 출처 정책/CORS 해결 필요)
* **사운드/버블**: Howler.js 등으로 수중 사운드·기포 파티클
* **스크린샷/레코딩**: canvas 캡처 API

---

## 접근성

* 마우스 클릭 시 텍스트 대비(흰 글자/반투명 검정 배경)
* 향후 키보드 포커스/ARIA 라이브 리전 추가 가능

---

## 브라우저 호환성

* 최신 Chromium/Firefox/Safari(2024+) 권장. WebGL2/ESM 지원 필요.

---

## 신뢰·책임·한계 고지

* **시뮬레이션 한계**: 유영은 보이드 휴리스틱 + 단순 경로 추종 기반으로, 수문역학적 정확성을 보장하지 않습니다.
* **자산/데이터 출처**: Three.js(UNPKG CDN) 외 외부 모델을 사용하지 않았습니다(모든 물체는 절차적 메시/셰이더).
* **검증 범위**: 포함된 `console.assert` 테스트는 **가시성·기본 동작** 위주로 작성되었습니다. 물리적/생태학적 타당성 검증은 포함되어 있지 않습니다.

---

## 라이선스/크레딧

* Three.js © MIT ([https://threejs.org](https://threejs.org))
* 코드/셰이더 샘플은 본 프로젝트 목적에 맞게 작성/변형되었습니다.

---

## 문의/요청

* 종별 모션/색감/크기, 경로, 식생 밀도 등 구체 파라미터를 알려주시면 프리셋으로 추가해 드립니다.
* HR/교육용 **강의 자료**로 사용할 수 있도록 도표/주석 버전도 제작 가능합니다.

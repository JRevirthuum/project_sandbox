# 💻 모바일 인앱 환경: 하드웨어 볼륨 감지 기능 구현 보고서

<img width="490" height="848" alt="image" src="https://github.com/user-attachments/assets/c55cc714-4f11-4a0a-91c6-d00955b6681e" />

## 🚧 프로젝트 개요

본 문서는 WebRTC 기반 화상 상담 솔루션의 고객 경험 향상을 위해, **하드웨어 볼륨 음소거 상태**를 웹 기술(Web Audio API)만으로 간접 감지하는 기능의 연구 및 구현 과정을 기록한다.
(with. Gemini)

### 1. 문제 정의 및 기술적 배경

#### 1.1. 해결하고자 했던 핵심 문제

고객이 상담 전 물리적인 하드웨어 볼륨을 0으로 설정하여 상담원의 목소리를 듣지 못하는 경우, 이는 곧바로 클레임으로 이어졌다. 표준 웹 API는 소프트웨어 볼륨은 제어하였으나, **하드웨어 볼륨 상태를 직접 읽을 수 있는 기능은 OS 보안 정책상 제공되지 않았다.**

#### 1.2. 이론적 접근: 어쿠스틱 피드백 루프 (Acoustic Feedback Loop)

하드웨어 볼륨 상태를 간접적으로 확인하기 위해 **루프백(Loopback)** 방식을 채택하였다.

1. **송출 (Controlled Output):** 사람이 듣기 힘든 **고주파 신호($\text{19kHz}$ Sine Wave)**를 **소프트웨어 볼륨** $\text{100\%}$ **상태**로 스피커를 통해 출력하도록 했다.

2. **감지 (Measured Input):** 마이크 스트림을 `AnalyserNode`에 연결하여 유입된 $\text{19kHz}$ 신호의 **에너지 레벨($\text{Level}$)**을 측정하였다.

3. **판단:** 소프트웨어적으로 최대로 설정했음에도 $\text{Level}$이 **임계값($\text{Threshold}=5$) 미만**일 경우, 이는 **하드웨어 볼륨**이 낮거나 음소거 상태임을 추정하는 근거로 삼았다.

### 2. 구현 과정 중 발생한 난관 (Challenges)

#### 2.1. 난관 1: 모바일 OS의 강력한 노이즈 캔슬링 (AEC)

* **현상:** PC에서는 정상 작동하였으나, 모바일 환경에서 하드웨어 볼륨을 최대로 해도 $\text{19kHz}$ 신호 레벨이 $\text{0} \sim \text{1}$로 극히 낮게 측정되었다.

* **원인:** 모바일 OS는 통화 품질을 위해 스피커 출력음을 마이크 입력에서 차단하는 **Acoustic Echo Cancellation (AEC)**을 강력하게 적용하고 있었다. 이는 웹 브라우저가 제어할 수 없는 **OS 커널 레벨** 기능이라고 판단했다.

#### 2.2. 난관 2: Level 값의 불안정성과 경고 깜빡임

* **현상:** $\text{Level}$ 값이 $\text{0} \rightarrow \text{11} \rightarrow \text{0}$ 등으로 빠르게 변동하며 경고 확정 조건을 순간적으로 만족해도, 팝업이 깜빡이거나 실행되지 않는 현상을 보였다.

* **원인:** `requestAnimationFrame`의 고빈도 루프 속도(약 60fps)와 시스템 간섭으로 인해 경고 확정까지 도달할 시간이 시스템적으로 보장되지 않았다.

### 3. 최종 해결책: 카운터 기반 안정화 로직 및 UI 제어

문제를 해결하기 위해 타이머 방식 대신 **프레임 카운터 방식**을 도입하고, $\text{CSS}$ 충돌을 방지하는 $\text{JS}$ 클래스 제어를 적용하였다.

#### 3.1. 카운터 기반 경고 확정

`draw()` 함수 루프 내에서 낮은 레벨($\text{Level} < 5$)이 **연속적으로 감지된 프레임 수**를 세어 신뢰성을 확보하였다.

| 조건 | 동작 | 결과 | 
 | ----- | ----- | ----- | 
| $\text{Level} < 5$ **지속** | `lowLevelCounter`를 증가시켰다 | $\text{120}$ 프레임 (2초) 도달 시 경고가 확정된다 | 
| $\text{Level} \ge 5$ **감지** | `lowLevelCounter`를 즉시 $\text{0}$으로 초기화하였다 | 경고 확정 전 신호가 복구되면 프로세스가 취소된다 | 

* **핵심 상수:** `const CONFIRMATION_FRAMES = 120;`

#### 3.2. UI 표시 안정화

JavaScript에서 인라인 스타일 대신 **CSS 클래스**를 토글하여 DOM 업데이트 충돌을 방지하였다.

* **함수:** `showVolumeWarning()`은 `$warningPopup.classList.remove('hidden');`을 실행하도록 변경했다.

* **CSS:** `#volumeWarningPopup.hidden { display: none; }` 정의를 활용했다.

### 4. 핵심 코드 요약 (안정화 로직)

```javascript
let lowLevelCounter = 0;
const CONFIRMATION_FRAMES = 120; // 2초 (60fps 기준)

function draw() {
    requestAnimationFrame(draw); 
    
    // ... Level 계산 로직 ...
    const level = micDataArray[targetIndex]; 

    if (level < VOLUME_THRESHOLD) {
        if (!isHardwareVolumeLow) {
            lowLevelCounter++; 
            if (lowLevelCounter >= CONFIRMATION_FRAMES) {
                isHardwareVolumeLow = true; 
                showVolumeWarning(); // 클래스 기반 표시
                console.log("[Counter] Volume Low Confirmed after 2 seconds.");
            }
        }
    } else {
        if (lowLevelCounter > 0) {
            lowLevelCounter = 0; // 카운터 초기화
            console.log("[Counter] Reset. Volume OK signal received.");
        }
        if (isHardwareVolumeLow) {
            isHardwareVolumeLow = false;
            hideVolumeWarning();
        }
    }
    // ... 시각화 로직 ...
}

function showVolumeWarning() {
    $warningPopup.classList.remove('hidden');
}
```

### 5. 최종 결론 및 권장 사항

이 방식은 네이티브 코드 없이 모바일 $\text{AEC}$ 환경을 **우회하여 하드웨어 볼륨 상태 변화를 감지하는 유일하고 현실적인 웹 기반 접근**이라고 결론지었다. 그러나 근본적인 $\text{AEC}$ 제어의 한계로 인해 **감지 신뢰도는 시스템 및 기기 환경에 따라 변동될 수 있다**고 보았다.

* **권장 사항:** 실제 서비스 통합 전, 다양한 모바일 기기(특히 노이즈 캔슬링 성능이 다른 기기)에서 `VOLUME_THRESHOLD` 및 `CONFIRMATION_FRAMES` 값을 미세 조정하는 QA 과정이 필수적이라고 권장하였다.

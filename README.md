# 🔐 STM32 기반 아날로그 다이얼 금고 시스템  
**Analog Dial Safe System using STM32 Nucleo-F411RE**
<br>
![Image](https://github.com/user-attachments/assets/de59dc97-d225-4cbf-b465-c10919eefa60)
<br>
STM32 Nucleo-F411RE 기반으로, 가변저항을 아날로그 다이얼로 활용하여 비밀번호를 입력하고,  
PWM 서보 모터로 물리적 잠금/해제를 수행하는 전자 금고 시스템입니다.  
비밀번호 입력부터 관리자 모드 전환, PW 변경까지 전 과정은 **FSM 기반 상태 제어 로직**으로 설계되어  
안정성, 보안성, 유지보수성을 모두 강화한 구조를 확보했습니다.

---
## 🔀 시스템 전체 흐름도 (Flowchart)
<br>
![mini_safe_flowchart](https://github.com/user-attachments/assets/c7af74a0-1346-46a2-84df-0e7c5a45c540.jpg)
<br>

---

## 🚀 주요 기능 (Key Features)
- **아날로그 다이얼 입력(ADC)** → 0\~4095 값을 10구간으로 나눠 0\~9 숫자 생성  
- **PWM 서보 제어** → 1500us(잠금), 2500us(해제)  
- **TACT Switch + EXTI 인터럽트** → 입력 확정 및 모드 전환  
- **200ms 소프트웨어 디바운싱** 적용 → 기계식 스위치 노이즈 제거 신뢰도 높은 입력
- **FSM 기반 4단계 관리자 인증 구조**
  - IDLE  
  - SUPERVISOR_INPUT  
  - CURRENT_INPUT  
  - NEW_PASSWORD_INPUT  

---

## 🛠 기술 스택 (Tech Stack)
| Category | Details |
|---------|---------|
| MCU | STM32 Nucleo-F411RE |
| Language | C (STM32 HAL) |
| Input | Potentiometer (ADC) |
| Output | Servo Motor (PWM) |
| UI | 16x2 I2C LCD |
| Interaction | TACT Switch × 2 (EXTI 인터럽트) |
| Architecture | FSM 기반 일반·관리자 상태 관리 |

---

## 🧩 시스템 구조 (System Architecture)
- **ADC** : 가변저 값 → 0\~9 숫자 변환  
- **PWM** : 서보 모터 각도 제어로 잠금/해제 수  
- **EXTI** : 버튼 입력 인터럽트 처리  
- **LCD** : 인증 진행 상태/숫자 실시간 표시  
- **FSM** : 안정적인 PW 검증·변경 흐름 제어  

---

## 📘 핵심 구현 (Core Implementation)

### ✔ FSM 기반 4단계 비밀번호 변경 인증 로직  
비밀번호 변경 과정의 **보안성과 안정성**을 확보하기 위해 상태를 명확히 분리하여 구현

| State (security_mode) | 설명 | TACT 2 입력 시 로직 |
|-----------------------|------|----------------------|
| **IDLE** | 기본 잠금/대기 상태 | SUPERVISOR_INPUT으로 전환 |
| **SUPERVISOR_INPUT** | 관리자 비밀번호 입력 단계 | 관리자 PW 검증 → 성공 시 CURRENT_INPUT 전환 |
| **CURRENT_INPUT** | 현재 금고 비밀번호 확인 단계 | PW 검증 → 성공 시 NEW_PASSWORD_INPUT 전환 |
| **NEW_PASSWORD_INPUT** | 새 비밀번호 입력 및 저장 | password[]에 저장(set_password()) 후 IDLE 복귀 |

### ✔ TACT 스위치 디바운싱 처리 (소프트웨어적 안정화)  
기계식 스위치의 **채터링(Chattering)** 문제를 소프트웨어 디바운싱으로 해결해 입력 신뢰성 강화
EXTI 인터럽트 발생 시, 200ms 이내의 중복 입력은 모두 무시

**구현 위치:** `HAL_GPIO_EXTI_Callback()`  

```c
if (now - last_press_time_tact1 > 200) {   // 200ms 디바운스
    handle_tact1();
    last_press_time_tact1 = now;
}
```

### ✔ 아날로그 다이얼 입력 → 숫자 변환 (ADC 안정화)

가변 저항의 아날로그 값(0\~4095)을 **ADC 인터럽트**로 읽고,  
이를 10개의 균등한 구간으로 나누어 **0\~9 정수(adc_val)** 로 매핑  
노이즈로 인한 값 불안정성을 제거해 **안정적인 비밀번호 입력** 가능

**구현 위치:** `HAL_ADC_ConvCpltCallback()`

```c
if (val < 200)      adc_val = 0;
else if (val < 500) adc_val = 1;
// ...
else if (val < 4000) adc_val = 8;
else                 adc_val = 9;
```

### ✔ PWM 기반 잠금/해제 제어

Timer 기반 PWM 신호로 서보 모터의 각도 정밀 제어  
비밀번호 검증 결과에 따라 서보 모터가 **잠금(Lock)** 또는 **해제(Unlock)** 위치로 이동

- **잠금(Lock)** : 1500us  
- **해제(Unlock)** : 2500us  

```c
__HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, 1500);  // Lock
__HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, 2500);  // Unlock
```
---

## 🐞 트러블슈팅 (Troubleshooting)

### 1) **ADC 입력 노이즈로 숫자 튐**
- 문제: 값 변동으로 숫자가 불안정  
- 해결: 10구간 고정 매핑 + 인터럽트 기반 안정화  
- 결과: 다이얼 움직임이 부드럽고 정확하게 표시됨  

### 2) **TACT 버튼 중복 입력 (채터링)**
- 문제: 버튼 1회 입력 → 다중 인터럽트 발생  
- 해결: 인터럽트 콜백에 200ms 디바운싱 적용  
- 결과: 입력 이벤트 정확히 1회만 처리됨  

### 3) **관리자 모드 PW 변경 과정에서 상태 꼬임**
- 문제: insert_num 초기화 누락 / FSM 흐름 불명확  
- 해결: 4단계 FSM 재정의 + 모든 분기에서 reset_nums() 호출  
- 결과: PW 변경이 안정적으로 작동  

---

## 🙋‍♂️ 담당 역할 (My Contribution)
- 아날로그 다이얼 기반 PW 입력 로직 전체 구현  
- ADC → 숫자 매핑 / insert_num 배열 구조 설계  
- 관리자 모드 포함 FSM 4단계 인증 시스템 개발  
- EXTI + 디바운싱(200ms) 버튼 로직 구현

---

## 📄 프로젝트 결과 및 배운 점
- FSM 구조 설계를 통한 확장 가능 시스템 구축
- ADC·PWM·EXTI·I2C 통합 제어 경험 강화
- 하드웨어 노이즈·채터링 등 실제 임베디드 문제 해결 능력 확보  
- HW/SW 통합 디버깅 능력 향상  

---

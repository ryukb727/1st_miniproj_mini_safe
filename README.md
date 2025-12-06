| [Korean 🇰🇷](#korean) | [Japanese 🇯🇵](#japanese) |
| :---: | :---: |

</div>

---

<div id="korean">

### 🇰🇷 Korean Version

# 🔐 STM32 기반 아날로그 다이얼 금고 시스템  
**Analog Dial Safe System using STM32 Nucleo-F411RE**
<br>
![Image](https://github.com/user-attachments/assets/de59dc97-d225-4cbf-b465-c10919eefa60)
<br>
![Image](https://raw.githubusercontent.com/ryukb727/1st_miniproj_mini_safe/refs/heads/main/images/mini_safe_flowchart.jpg)
<br>

---

## 💡 1. 프로젝트 개요
STM32 Nucleo-F411RE 기반으로, 가변저항을 아날로그 다이얼로 활용하여 비밀번호를 입력하고, PWM 서보 모터로 물리적 잠금/해제를 수행하는 전자 금고 시스템입니다.  
비밀번호 입력부터 관리자 모드 전환, 비밀번호 변경에 이르는 전 과정은 **FSM 기반 상태 제어 로직**으로 설계하여 안정성, 보안성, 유지보수성을 모두 강화한 구조를 확보했습니다.

### 📍 핵심 목표
- 아날로그 입력(ADC) 기반 금고 비밀번호 입력 시스템 구현
- PWM 서보 모터를 이용한 잠금/해제 메커니즘 제어
- 관리자 모드 도입으로 보안 기능 확장
- FSM 기반 상태 전환 구조로 시스템 안정성 확보

---

## 🚀 2. 주요 기능
- **아날로그 다이얼 입력(ADC)** → 0\~4095 값을 10구간으로 나눠 0\~9 숫자 생성  
- **PWM 서보 모터 제어** → 1500us(잠금), 2500us(해제)  
- **TACT Switch + EXTI 인터럽트** → 입력 확정 및 모드 전환  
- **200ms 소프트웨어 디바운싱** 적용 → 기계식 스위치의 노이즈를 제거해 높은 입력 신뢰도
- **FSM 기반 4단계 관리자 인증 구조**
  - IDLE  
  - SUPERVISOR_INPUT  
  - CURRENT_INPUT  
  - NEW_PASSWORD_INPUT  

---

## 🛠 3. 개발 환경 및 기술 스택
![STM32](https://img.shields.io/badge/MCU-STM32F411RE-03234B?style=for-the-badge&logo=stmicroelectronics&logoColor=white)
![C](https://img.shields.io/badge/Language-C-00599C?style=for-the-badge&logo=c&logoColor=white)
![HAL](https://img.shields.io/badge/STM32-HAL%20Library-03234B?style=for-the-badge&logo=stmicroelectronics&logoColor=white)
![ADC](https://img.shields.io/badge/Input-ADC%20(Potentiometer)-A2C93A?style=for-the-badge)
![PWM](https://img.shields.io/badge/Output-PWM%20Servo%20Motor-FFB400?style=for-the-badge)
![LCD](https://img.shields.io/badge/I2C-16x2%20LCD-1E90FF?style=for-the-badge)
![TACT](https://img.shields.io/badge/Interaction-TACT%20Switch%20×%202-555555?style=for-the-badge)
![EXTI](https://img.shields.io/badge/Interrupt-EXTI%20(External%20Interrupt)-8A2BE2?style=for-the-badge)
![FSM](https://img.shields.io/badge/Architecture-FSM%20(State%20Machine)-4CAF50?style=for-the-badge)

---

## 🧩 4. 시스템 구조
- **ADC** : 가변저항 값 → 0\~9 숫자 변환  
- **PWM** : 서보 모터 각도 제어로 잠금/해제 수행  
- **EXTI** : 버튼 입력 인터럽트 처리  
- **LCD** : 인증 진행 상태/숫자 실시간 표시  
- **FSM** : 안정적인 비밀번호 검증·변경 흐름 제어  

---

## 📘 5. 핵심 구현

### 1) FSM 기반 4단계 비밀번호 변경 인증 로직  
비밀번호 변경 과정의 **보안성과 안정성**을 확보하기 위해 상태를 명확히 분리하여 구현

| State (security_mode) | 설명 | TACT 2 입력 시 로직 |
|-----------------------|------|----------------------|
| **IDLE** | 기본 잠금/대기 상태 | SUPERVISOR_INPUT으로 전환 |
| **SUPERVISOR_INPUT** | 관리자 비밀번호 입력 단계 | 관리자 PW 검증 → 성공 시 CURRENT_INPUT 전환 |
| **CURRENT_INPUT** | 현재 금고 비밀번호 확인 단계 | PW 검증 → 성공 시 NEW_PASSWORD_INPUT 전환 |
| **NEW_PASSWORD_INPUT** | 새 비밀번호 입력 및 저장 | password[]에 저장(set_password()) 후 IDLE 복귀 |

### 2) TACT 스위치 디바운싱 처리 (소프트웨어적 안정화)  
기계식 스위치의 **채터링(Chattering)** 문제를 소프트웨어 디바운싱으로 해결해 입력 신뢰성 강화
<br>
인터럽트 발생 시, 200ms 이내의 중복 입력은 모두 무시

**구현 위치:** `HAL_GPIO_EXTI_Callback()`  

```c
if (now - last_press_time_tact1 > 200) {   // 200ms 디바운스
    handle_tact1();
    last_press_time_tact1 = now;
}
```

### 3) 아날로그 다이얼 입력 → 숫자 변환 (ADC 안정화)

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

### 4) PWM 기반 잠금/해제 제어

Timer 기반 PWM 신호로 서보 모터의 각도 정밀 제어  
비밀번호 검증 결과에 따라 서보 모터가 **잠금(Lock)** 또는 **해제(Unlock)** 위치로 이동

- **잠금(Lock)** : 1500us  
- **해제(Unlock)** : 2500us  

```c
__HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, 1500);  // Lock
__HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, 2500);  // Unlock
```
---

## 🐞 6. 트러블슈팅

### 1) **ADC 입력 노이즈로 숫자 튐**
- 문제: 값 변동으로 숫자가 불안정  
- 해결: 10구간 고정 매핑 + 인터럽트 기반 안정화  
- 결과: 다이얼 움직임이 부드럽고 정확하게 표시됨  

### 2) **TACT 버튼 중복 입력 (채터링)**
- 문제: 버튼 1회 입력 → 다중 인터럽트 발생  
- 해결: 인터럽트 콜백에 200ms 디바운싱 적용  
- 결과: 입력 이벤트 정확히 1회만 처리됨  

### 3) **관리자 모드 비밀번호 변경 과정에서 상태 꼬임**
- 문제: insert_num 초기화 누락 / FSM 흐름 불명확  
- 해결: 4단계 FSM 재정의 + 모든 분기에서 reset_nums() 호출  
- 결과: 안정적인 비밀번호 변경 성공  

---

## 🙋‍♂️ 7. 담당 역할
- 아날로그 다이얼 기반 비밀번호 입력 로직 전체 구현  
- ADC → 숫자 매핑 / insert_num 배열 구조 설계  
- 관리자 모드 포함 FSM 4단계 인증 시스템 개발  
- EXTI + 디바운싱(200ms) 버튼 로직 구현

---

## 📄 8. 프로젝트 결과 및 배운 점
- FSM 구조 설계를 통한 확장 가능 시스템 구축
- ADC·EXTI·I2C 통합 제어 경험 강화
- 하드웨어 노이즈·채터링 등 실제 임베디드 문제 해결 능력
- HW/SW 통합 디버깅 능력 향상  

---
#### 📖 프로젝트 노션 URL
https://www.notion.so/hiawath/7-242c59623e6180f1a17bd6faf05222ce

---

<div align="center">
<a href="#japanese">⬇️ 日本語バージョンへ移動 (Go to Japanese Version) ⬇️</a>
</div>

</div>

---

<div id="japanese">

### 🇯🇵 Japanese Version

# 🔐 STM32ベース アナログダイヤル金庫システム
Analog Dial Safe System using STM32 Nucleo-F411RE
<br>
![Image](https://github.com/user-attachments/assets/de59dc97-d225-4cbf-b465-c10919eefa60)
<br>
![Image](https://raw.githubusercontent.com/ryukb727/1st_miniproj_mini_safe/refs/heads/main/images/mini_safe_flowchart.jpg)
<br>


## 💡 1. プロジェクト概要

本プロジェクトは、STM32 Nucleo-F411REをベースとし、可変抵抗器をアナログダイヤルとして活用して暗証番号を入力し、PWMサーボモーターで物理的なロック/解除を行う電子金庫システムです。  
暗証番号入力から管理者モードへの切り替え、パスワード変更に至る全過程は、**FSM (有限状態機械) ベースの状態制御ロジック**で設計されており、安定性、セキュリティ、保守性を強化した構造を確保しています。

### 📍 主要目標

- アナログ入力（ADC）に基づいた金庫暗証番号入力システムの実装
- PWMサーボモーターを用いたロック/解除メカニズムの制御
- 管理者モードの導入によるセキュリティ機能の拡張
- FSMベースの状態遷移構造によるシステム安定性の確保

---

## 🚀 2. 主要機能

- **アナログダイヤル入力 (ADC)** → 0～4095の値を10区間に分けて0～9の数字を生成
- **PWMサーボモーター制御** → 1500us（ロック）、2500us（解除）
- **TACTスイッチ + EXTI割り込み** → 入力確定およびモード切り替え
- **200ms ソフトウェアデバウンスの適用** → 機械式スイッチのノイズを除去し、信頼性の高い入力
- **FSMベースの4段階管理者認証構造**
  - IDLE
  - SUPERVISOR_INPUT
  - CURRENT_INPUT
  - NEW_PASSWORD_INPUT

---

## 🛠 3. 開発環境および技術スタック

![STM32](https://img.shields.io/badge/MCU-STM32F411RE-03234B?style=for-the-badge&logo=stmicroelectronics&logoColor=white)
![C](https://img.shields.io/badge/Language-C-00599C?style=for-the-badge&logo=c&logoColor=white)
![HAL](https://img.shields.io/badge/STM32-HAL%20Library-03234B?style=for-the-badge&logo=stmicroelectronics&logoColor=white)
![ADC](https://img.shields.io/badge/Input-ADC%20(Potentiometer)-A2C93A?style=for-the-badge)
![PWM](https://img.shields.io/badge/Output-PWM%20Servo%20Motor-FFB400?style=for-the-badge)
![LCD](https://img.shields.io/badge/I2C-16x2%20LCD-1E90FF?style=for-the-badge)
![TACT](https://img.shields.io/badge/Interaction-TACT%20Switch%20×%202-555555?style=for-the-badge)
![EXTI](https://img.shields.io/badge/Interrupt-EXTI%20(External%20Interrupt)-8A2BE2?style=for-the-badge)
![FSM](https://img.shields.io/badge/Architecture-FSM%20(State%20Machine)-4CAF50?style=for-the-badge)

---

## 🧩 4. システム構造

- **ADC** : 可変抵抗値 → 0～9の数字変換
- **PWM** : サーボモーターの角度制御によるロック/解除
- **EXTI** : ボタン入力の割り込み処理
- **LCD** : 認証進行状態/数字のリアルタイム表示
- **FSM** : 安定したパスワード検証・変更フローの制御

---

## 📘 5. 主要な実装

### 1) FSMベース 4段階パスワード変更認証ロジック

パスワード変更過程の**セキュリティと安定性**を確保するために、状態を明確に分離して実装

| State (security_mode) | 説明                      | TACT 2 入力時のロジック                          |
|-----------------------|---------------------------|--------------------------------------------------|
| **IDLE**              | 基本的なロック/待機状態    | SUPERVISOR_INPUTへ遷移                           |
| **SUPERVISOR_INPUT**  | 管理者パスワード入力段階  | 管理者PWを検証 → 成功時にCURRENT_INPUTへ遷移  |
| **CURRENT_INPUT**     | 現在の金庫パスワード確認段階 | PWを検証 → 成功時にNEW_PASSWORD_INPUTへ遷移 |
| **NEW_PASSWORD_INPUT**| 新しいパスワードの入力および保存 | password[]に保存(set_password())後、IDLEへ復帰 |

### 2) TACTスイッチのデバウンス処理 (ソフトウェア的安定化)

機械式スイッチの**チャタリング (Chattering)** 問題を**ソフトウェアデバウンス**で解決し、入力の信頼性を強化  
EXTI(外部割り込み)発生時、200ms以内の重複入力はすべて無視

実装箇所: HAL_GPIO_EXTI_Callback()

```c
if (now - last_press_time_tact1 > 200) {   // 200ms デバウンス
    handle_tact1();
    last_press_time_tact1 = now;
}
```

### 3) アナログダイヤル入力 → 数字変換 (ADC安定化)

可変抵抗のアナログ値 (0～4095) を**ADC割り込み**で読み取り、これを10個の均等な区間に分けて**0～9の整数 (adc_val)** にマッピング  
ノイズによる値の不安定性を除去し、**暗証番号入力を安定化**

実装箇所: HAL_ADC_ConvCpltCallback()

```c
if (val < 200)      adc_val = 0;
else if (val < 500) adc_val = 1;
// ...
else if (val < 4000) adc_val = 8;
else                 adc_val = 9;
```

### 4) PWMベースのロック/解除制御

タイマーベースのPWM信号でサーボモーターの角度を精密に制御  
暗証番号の検証結果に基づき、サーボモーターが**ロック (Lock)** または **解除 (Unlock)** 位置へ移動

ロック (Lock) : 1500us  
解除 (Unlock) : 2500us

```c
__HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, 1500);  // Lock
__HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, 2500);  // Unlock
```
---

## 🐞 6. トラブルシューティング

### 1) ADC入力ノイズによる数字の飛び
- **問題**: 値の変動により数字が不安定
- **解決策**: 10区間の固定マッピング + 割り込みベースの安定化
- **結果**: ダイヤルの動きがスムーズかつ正確に表示される

### 2) TACTボタンの重複入力 (チャタリング)
- **問題**: ボタン1回入力 → 複数回の割り込みが発生
- **解決策**: 割り込みコールバックに200msのデバウンスを適用
- **結果**: 入力イベントが正確に1回のみ処理される

### 3) 管理者モードのパスワード変更プロセスにおける状態の混乱
- **問題**: insert_numの初期化漏れ / FSMフローの不明確さ
- **解決策**: 4段階FSMの再定義 + すべての分岐でreset_nums()を呼び出し
- **結果**: パスワード変更が安定して動作する

---

## 🙋‍♂️ 7. 担当役割

- アナログダイヤルベースのPW入力ロジック全体の実装
- ADC → 数字マッピング / insert_num配列構造の設計
- 管理者モードを含むFSM 4段階認証システムの開発
- EXTI + デバウンス (200ms) ボタンロジックの実装

---

## 📄 8. プロジェクト結果と学んだこと

- FSM構造設計による拡張可能なシステム構築
- ADC・EXTI・I2Cの統合制御経験の強化
- ハードウェアノイズ・チャタリングなどの実際的な組み込み問題解決能力
- HW/SW統合デバッグ能力の向上

---

## 📖 プロジェクト Notion URL
[プロジェクト Notion](https://www.notion.so/hiawath/7-242c59623e6180f1a17bd6faf05222ce)

---

<div align="center">
<a href="#korean">⬆️ 한국어 버전으로 돌아가기 (Go back to Korean Version) ⬆️</a>
</div>

</div>

---

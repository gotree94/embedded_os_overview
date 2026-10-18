# 임베디드 운영체제 한눈에 보기

> 출처: YouTube "Every Embedded Operating System Explained in 9 Minutes" (https://youtu.be/R7dhNELFc9A) 자막 기반 정리 <br>
> 자막의 음성 인식 오류(예: VEX Works → VxWorks, Free Arts → FreeRTOS, Zephr → Zephyr)는 바로잡아 반영했습니다.

---

## 0. 학습 목표

- 임베디드 시스템에서 사용하는 6가지 소프트웨어 실행 환경의 **구조·특징·장단점**을 설명할 수 있다.
- **프로젝트 요구사항**(실시간성, 안전 인증, 비용, 연산 성능)에 따라 적절한 OS를 선택할 수 있다.
- 각 OS의 대표 적용 사례를 제품·산업 분야와 연결할 수 있다.

---

## 1. 전체 개요

임베디드 소프트웨어는 프로젝트 복잡도가 올라갈수록 아래 순서로 단계적으로 진화합니다.

```
Bare Metal  →  FreeRTOS  →  Zephyr  →  (고신뢰) VxWorks / QNX
                                    →  (고성능 AP) Embedded Linux
```

| 구분 | 등장 | 유형 | 라이선스 | 대상 하드웨어 | 실시간성 |
|---|---|---|---|---|---|
| Bare Metal | 1970년대 | OS 없음 (Super Loop) | - | 8/32비트 MCU | 마이크로초급 지연 |
| FreeRTOS | 2003 | 선점형 RTOS 커널 | 오픈소스 | MCU | 결정적(Deterministic) |
| VxWorks | 1987 | 모놀리식 상용 RTOS | 상용(고가) | MCU/프로세서 | Hard RT, 마이크로초급 |
| Embedded Linux | 1991(Linux 커널) | 범용 OS 기반 | 오픈소스 | Cortex-A급 AP | Soft RT (PREEMPT_RT 패치 시 개선) |
| Zephyr | 2016 | 모듈형 RTOS 플랫폼 | 오픈소스 | MCU (ARM Cortex-M, RISC-V, ESP32) | RTOS급 |
| QNX | 1980 | 마이크로커널 상용 RTOS | 상용(고가) | 고성능 SoC | Hard RT |

---

## 2. Bare Metal (OS 없음)

### 개념
- OS, 스케줄러 등 모든 추상화 계층을 제거하고 **하드웨어 레지스터를 직접 제어**하는 방식.
- 전체 코드가 하나의 `while(1)` **Super Loop** 안에서 동작하며, 외부 이벤트는 **하드웨어 인터럽트**로 처리.
- 1970년대 초기 마이크로컨트롤러와 함께 등장.

### 장점
- 커널·스케줄러가 없어 **오버헤드 0**, ROM/Flash 100%를 애플리케이션에 사용
- 전원 인가 즉시 실행, **최소 전력**, 마이크로초 단위 응답 지연

### 단점
- 복잡한 다단계 상태 머신을 하나의 루프에 넣으면 코드가 비대해지고 유지보수가 어려움
- 느린 센서 읽기나 외부 버스 응답을 기다리며 한 함수가 멈추면 **시스템 전체가 정지** → 모든 주변장치를 Non-blocking 방식으로 작성해야 함
- 파일시스템, 네트워크 드라이버, **메모리 보호 기능 없음** → 저수준 드라이버를 직접 작성/이식

### 대표 적용 사례
전동칫솔, TV 리모컨, 전동공구 안전 인터락, 저가 센서 노드, 저전력 차량 제어 모듈

### 핵심 코드 구조

```c
int main(void)
{
    hw_init();
    while (1) {            // Super Loop
        read_sensors();
        update_state_machine();
        drive_outputs();
    }
}

void EXTI0_IRQHandler(void)   // 하드웨어 인터럽트
{
    // 이벤트 처리 (짧게!)
}
```

---

## 3. FreeRTOS (MCU용 오픈소스 RTOS)

### 개념
- 2003년 Richard Barry가 개발, **2017년 Amazon Web Services가 인수**.
- 전 세계에서 **가장 널리 배포된 오픈소스 MCU용 RTOS**.
- 애플리케이션을 여러 **독립 Task**로 분리하고, 작은 **선점형(Preemptive) 커널**이 관리.

### 동작 원리
- 스케줄러가 Task **우선순위**를 계속 감시하고 **Context Switching**을 수행
- 우선순위가 높은 Task는 필요한 순간 즉시 실행 (결정적 타이밍)

### 장점
- 커널 크기 약 **4~9KB Flash**로 매우 작음
- ST, Espressif, NXP, Microchip 등 **주요 칩 벤더 SDK에 기본 통합**

### 단점
- 기본 설정에서는 모든 Task가 **하나의 보호되지 않는 메모리 공간**을 공유 → 로깅 Task의 Null 포인터나 스택 오버플로가 전체 시스템을 다운시킬 수 있음
- 기본은 "실행기(Executor)"에 불과 → 네트워크, 파일시스템, 암호화 스택은 **서드파티 라이브러리로 별도 통합** 필요

### 대표 적용 사례
ESP32 스마트홈 허브, 웨어러블 활동 추적기, 스마트 전력/수도 계량기, 산업용 센서 트랜스미터

### 핵심 코드 구조

```c
void vSensorTask(void *pv)
{
    for (;;) {
        read_sensor();
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}

int main(void)
{
    xTaskCreate(vSensorTask, "sensor", 256, NULL, 3, NULL);
    xTaskCreate(vLogTask,    "log",    256, NULL, 1, NULL);
    vTaskStartScheduler();
}
```

---

## 4. VxWorks (항공우주·방산 임무 필수 RTOS)

### 개념
- 1987년 **Wind River Systems** 출시. **상용 임무 필수(Mission-Critical) RTOS 시장의 선두**.
- 고성능 **모놀리식 실시간 커널** 구조. 오류가 허용되지 않는 환경 대상.

### 특징
- **Hard Real-Time**, 마이크로초급 지연, 최소 타이밍 지터
- 사전 패키징된 **안전 인증 산출물** 제공
  - **DO-178C**: 상업용 항공전자
  - **IEC 61508**: 산업 자동화
  - **ISO 26262**: 자동차 기능안전
- 고급 **메모리 보호**로 불량 Task가 핵심 동작을 흔들지 못하게 방지

### 단점
- 고가의 독점 상용 라이선스 → 취미/저마진 소비자 전자제품에는 비현실적
- 커스텀 기능·드라이버가 **Wind River의 폐쇄적(Vendor Lock-in) 생태계**에 종속

### 대표 적용 사례
NASA Mars Perseverance 로버, Boeing 비행제어 컴퓨터, 상용 MRI 장비, 철도 제어 시스템, 고속철도 신호 장비

---

## 5. Embedded Linux (애플리케이션 프로세서용 풀 OS)

### 개념
- MCU의 연산 여력이 한계에 달하면 **애플리케이션 프로세서**로 이동 → Embedded Linux 영역.
- 1991년 Linus Torvalds의 Linux 커널에서 출발.
- Ubuntu 같은 **데스크톱 배포판이 아님**. **Buildroot**나 **Yocto Project**로 우리 하드웨어에 맞춘 **최소 커널 + 루트 파일시스템**을 직접 빌드.

### 장점
- 거대한 소프트웨어 생태계: **Python, C++, Rust**, 관계형 DB, 웹 API, **Qt/TouchGFX** 등 그래픽 프레임버퍼
- USB, 터치 디스플레이, 플래시 스토리지, 고속 네트워크용 **검증된 드라이버**가 풍부

### 단점
- 기본 스케줄링은 **Soft Real-Time / 비결정적** (`PREEMPT_RT` 패치로 개선 가능)
- **ARM Cortex-A급** 32/64비트 프로세서, 수 MB~GB 단위 RAM 필요
- 부팅 시간이 **밀리초가 아닌 초 단위**

### 대표 적용 사례
스마트 TV UI, EV 충전기 단말, 차량 인포테인먼트 터치스크린, 네트워크 라우터, 산업용 게이트웨이 HMI

### 빌드 도구 비교

| 도구 | 특징 |
|---|---|
| Buildroot | 단순하고 빠름, 소규모 이미지에 적합 |
| Yocto Project | 레이어 기반, 대규모·장기 제품 관리에 적합 |

---

## 6. Zephyr RTOS (현대적 IoT 플랫폼)

### 개념
- 단순 Task 라이브러리가 아닌 **현대적 소프트웨어 플랫폼**에 가까운 RTOS.
- **2016년 Linux Foundation** 산하에서 출범. Nordic Semiconductor, NXP, Meta 등이 후원.
- 자원이 제한된 IoT 개발에 **통일된 구조와 표준화**를 제공.

### 특징
- **Linux에서 영감받은 설계**: **Device Tree**로 하드웨어 핀맵을 C 소스와 분리, 모듈형 **Kconfig** 빌드 시스템
- **즉시 사용 가능한 프로덕션급 프로토콜 스택**: Bluetooth Low Energy, Wi-Fi, Thread, **Matter**
- 같은 애플리케이션 코드를 **ARM Cortex-M, RISC-V, ESP32** 등에 재컴파일 가능
- **MPU(Memory Protection Unit)** 기반 하드웨어 수준 메모리 보호

### 단점
- CMake + Device Tree + Kconfig 설정으로 **FreeRTOS 대비 학습 곡선이 가파름**
- Flash 사용량이 큼: 기본 구성만도 **20~50KB 이상**

### 대표 적용 사례
BLE 자산 추적기, 스마트 헬스 모니터, **Nordic nRF52/nRF53** 기반 Matter 인증 스마트홈 기기

### 구조 예시

```
app/
├── CMakeLists.txt
├── prj.conf          # Kconfig 옵션
├── boards/
│   └── my_board.overlay   # Device Tree overlay
└── src/main.c
```

---

## 7. QNX Neutrino (자동차 안전용 마이크로커널)

### 개념
- 1980년 **Quantum Software Systems** 개발, **2010년 BlackBerry 인수**.
- 진정한 **마이크로커널(Microkernel)** 구조의 상용 RTOS.
- 모놀리식 OS는 드라이버·파일시스템·네트워크 스택이 **하나의 커널 메모리 공간**에서 돌지만, QNX는 마이크로커널에 **스케줄링과 프로세스 간 통신(IPC)** 같은 필수 기능만 두고 **나머지는 격리된 User Space 프로세스**로 실행.

### 장점
- **탁월한 결함 격리**: 3D 그래픽 드라이버나 미디어 플레이어가 죽어도 마이크로커널이 **해당 프로세스만 수 ms 내 재시작**, 핵심 안전 기능은 영향 없음
- Hard Real-Time 성능 + 최고 수준의 **ISO 26262 (ASIL D)** 기능안전 인증

### 단점
- 높은 상용 라이선스 비용 → 엔터프라이즈 예산 한정
- 마이크로커널 IPC로 인해 모놀리식 커널 대비 **Context Switching 오버헤드 소폭 증가**

### 대표 적용 사례
자동차 디지털 콕핏, ADAS(첨단 운전자 보조), 자동화 조립 로봇 — **2억 3천만 대 이상** 차량에 탑재

### 모놀리식 vs 마이크로커널

| 구분 | 모놀리식 (VxWorks, Linux) | 마이크로커널 (QNX) |
|---|---|---|
| 드라이버/FS/네트워크 | 커널 공간 | **User Space 프로세스** |
| 장애 영향 | 커널 전체로 확산 가능 | **해당 프로세스만 재시작** |
| 성능 | IPC 오버헤드 적음 | IPC 오버헤드 소폭 증가 |

---

## 8. 종합 비교

| 항목 | Bare Metal | FreeRTOS | VxWorks | Embedded Linux | Zephyr | QNX |
|---|---|---|---|---|---|---|
| 커널 구조 | 없음 | 선점형 소형 커널 | 모놀리식 | 모놀리식 | 모듈형 RTOS | 마이크로커널 |
| 메모리 풋프린트 | 최소 | 4~9KB | 중~대 | MB~GB급 | 20~50KB+ | 중~대 |
| 실시간성 | 마이크로초 | 결정적 | Hard RT | Soft RT | RTOS급 | Hard RT |
| 메모리 보호 | 없음 | 기본 없음 | 있음 | 있음(MMU) | MPU 지원 | 있음(프로세스 격리) |
| 안전 인증 | - | 제한적 | DO-178C, IEC 61508, ISO 26262 | 별도 노력 필요 | 진행 중 | ISO 26262 최고 등급 |
| 비용 | 무료 | 무료 | 고가 | 무료 | 무료 | 고가 |
| 학습 난이도 | 낮음(초기)~높음(규모 커질수록) | 낮음 | 중~높음 | 높음 | 높음 | 높음 |
| 대표 분야 | 가전, 저가 센서 | IoT, 웨어러블 | 항공·우주·방산·의료 | 인포테인먼트, 게이트웨이 | BLE, Matter | 자동차 콕핏, ADAS |

---

## 9. 선택 가이드

```
Q1. 안전 인증(항공/자동차/의료)이 필수인가?
    ├─ 예 → 예산 있음 → VxWorks (항공·방산·의료·철도) / QNX (자동차)
    └─ 아니오 ↓
Q2. Cortex-A급 AP + 풍부한 SW 생태계(GUI, DB, 네트워크)가 필요한가?
    ├─ 예 → Embedded Linux (Buildroot / Yocto)
    └─ 아니오 ↓
Q3. 동시에 여러 작업을 처리해야 하는가?
    ├─ 아니오 (단순 제어) → Bare Metal
    └─ 예 ↓
Q4. BLE/Thread/Matter 등 통합 프로토콜 스택과 멀티 아키텍처 이식성이 중요한가?
    ├─ 예 → Zephyr
    └─ 아니오 → FreeRTOS
```

---

## 10. 강사 참고 노트

영상은 9분 분량의 개요이므로 일부를 단순화합니다. 수업 시 보완하면 좋은 포인트입니다.

- **FreeRTOS 메모리 보호**: "기본 설정은 보호 없음"이 맞지만, MPU 포트(FreeRTOS-MPU)와 AWS의 네트워크·보안 라이브러리(coreMQTT, TLS 등)도 존재합니다.
- **Bare Metal vs RTOS 경계**: 요즘은 RTOS 없이도 인터럽트 + 상태 머신 + 협력형 스케줄러로 상당한 복잡도를 감당하기도 합니다.
- **Linux 실시간성**: `PREEMPT_RT`가 메인라인에 통합되는 흐름이므로 최신 커널 버전 기준으로 다시 확인하면 좋습니다.
- **안전 인증 주의**: "인증 산출물 제공"은 OS가 인증된다는 의미이며, **최종 제품의 인증은 별도**로 받아야 합니다.
- 영상의 수치(4~9KB, 20~50KB, 2억 3천만 대 등)는 **대략적 수치**로 설정·버전·시점에 따라 달라집니다.

---

## 11. 복습 퀴즈

1. Bare Metal에서 한 함수가 느린 센서 응답을 기다리면 어떤 문제가 생기는가?
2. FreeRTOS에서 Task 하나의 스택 오버플로가 전체 시스템에 영향을 줄 수 있는 이유는?
3. VxWorks가 항공·의료·철도 분야에서 선호되는 이유 두 가지는?
4. Embedded Linux가 기본적으로 Hard Real-Time이 아닌 이유와 해결 방법은?
5. Zephyr에서 Device Tree와 Kconfig는 각각 어떤 역할을 하는가?
6. QNX에서 그래픽 드라이버가 죽어도 핵심 안전 기능이 유지되는 이유는?
7. 다음 제품에 적합한 OS를 고르고 이유를 설명하시오.
   - (a) 전동칫솔  (b) Nordic nRF53 기반 Matter 스마트 조명  (c) 차량 디지털 콕핏  (d) EV 충전기 터치 단말

### 예시 답안
1. 루프 전체가 멈춰 다른 모든 기능이 정지 (Non-blocking 설계 필요)
2. 모든 Task가 보호되지 않는 단일 메모리 공간을 공유하기 때문
3. Hard Real-Time 성능과 DO-178C / IEC 61508 / ISO 26262 등 안전 인증 산출물 제공
4. 스케줄링이 Soft RT/비결정적이기 때문. `PREEMPT_RT` 패치로 개선
5. Device Tree: 하드웨어(핀맵 등) 기술, Kconfig: 기능·모듈 구성 선택
6. 드라이버가 User Space 프로세스로 격리되어 마이크로커널이 해당 프로세스만 재시작하기 때문
7. (a) Bare Metal (b) Zephyr (c) QNX (d) Embedded Linux

---

## 12. 핵심 요약

| OS | 한 줄 요약 |
|---|---|
| **Bare Metal** | 가장 가볍고 빠르지만 복잡해지면 감당 불가 |
| **FreeRTOS** | 작고 널리 쓰이는 MCU RTOS, 스택은 직접 조립 |
| **VxWorks** | 항공·방산·의료용 상용 Hard RT, 인증 산출물 강점 |
| **Embedded Linux** | 풍부한 생태계, 대신 무겁고 Soft RT |
| **Zephyr** | 표준화·프로토콜 스택 내장·이식성, 학습 곡선이 높음 |
| **QNX** | 마이크로커널 결함 격리, 자동차 안전의 강자 |

# Eye_Q_System
Vision Ai 활용 자동화 시스템

요구사항 분석 - 설계 - 구현 - 테스트 - 유지보수

#요구사항 분석

EYE_Q_SYSTEM (Vision AI활용 마킹시스템)

1. 컨베이어가동 (인버터_485통신으로 제어)- WPF사용 예정
2. 제품 픽엔플레이스 ( 다관절 or 스카라)
3. 머신비전( 제품,객체탐지 Yolo Seg, OpenCV)
4. 비전에 제품 객체 탐지되면 인버터 정지
5. 마킹을 해도되는지 양품확인 (OK,NG판별)
6. NG 일 경우 어떤 불량인지 확인후 DB에 저장
7. NG인 경우 Airblow로 날림
8. OK 이면 OpenCV로 경계검출 
-센터확인-수평확인(몇도 돌아갔는지) 
(*실제 장비가 마킹하는 위치와, 픽셀상 위치가 다를수 있다.)
9. 레이져 마킹 진행
(검출 데이터를 마킹기에 어떻게 전달하지?)
10. 마킹이 제대로 되었는지확인 (Vision AI활용, Yolo v8)
11. NG는  Airblow처리
12. OK는 박스에 보관 후 + 추가 될때마다 +1씩
13. OK와 NG 수량은 데이터 베이스에 저장(MSsql, MariaDB, MySql, Postgre)
14. 완성?


---

## 설계

# Eye_Q 시스템 설계 문서

문서 버전: v0.1  
작성 기준: 요구사항 분석 완료 후 설계 단계 진입  
목적: 구현 단계에서 기능 누락이 발생하지 않도록 요구사항, 기능, 화면, DB, API, 장비통신, 예외처리, 테스트 항목을 연결하여 관리한다.

---

## 1. 설계 단계 목표

Eye_Q 시스템의 설계 단계 목표는 단순한 화면 구성이나 코드 구조 결정이 아니다.

본 설계 단계의 핵심 목적은 다음과 같다.

> 요구사항 → 기능 → 화면 → DB → API → 장비통신 → 예외처리 → 로그 → 테스트 항목까지 전부 연결하여 구현 누락을 방지한다.

따라서 구현 전 반드시 다음 산출물을 확정한다.

| 번호 | 설계 산출물 | 목적 |
|---|---|---|
| 1 | 시스템 전체 구조도 | PC, 카메라, 컨베이어, 인버터, 마킹기, DB, 웹 화면 연결 관계 확정 |
| 2 | 기능 명세서 | 구현해야 할 기능을 빠짐없이 분해 |
| 3 | 화면 설계서 | WPF/Vue 화면에서 무엇을 보여줄지 확정 |
| 4 | DB 설계서 | 검사 결과, 마킹 결과, 장비 로그, 알람 저장 구조 확정 |
| 5 | API 설계서 | WPF, Vue, Backend 간 데이터 이동 방식 확정 |
| 6 | 장비 통신 설계서 | RS485 Modbus, G100C 인버터, 레이저 마킹기 명령 구조 확정 |
| 7 | 상태 흐름도 | 자동운전/정지/검사/마킹/알람 상태 흐름 확정 |
| 8 | 예외처리표 | 카메라 실패, 통신 실패, 물체 미검출, 마킹 실패 등 대응 확정 |
| 9 | 테스트 체크리스트 | 구현 후 누락 여부 검증 |

---

## 2. Eye_Q 시스템 개요

Eye_Q 시스템은 컨베이어 위 제품을 카메라로 검출하고, 제품의 위치와 각도를 계산한 후, 컨베이어를 정지시키고 레이저 마킹기에 각인 명령을 전달하는 자동 비전 기반 마킹 시스템이다.

### 2.1 핵심 공정 흐름

```text
1. 컨베이어 구동
↓
2. 카메라 화면에서 물체 5~10개 검출
↓
3. 물체 감지 후 RS485 Modbus로 G100C 인버터 정지 지령 전송
↓
4. OpenCV로 경계 검출
↓
5. 중심점 계산
↓
6. 각도 계산
↓
7. 수평 여부 판단
↓
8. 실제 마킹 위치 보정
↓
9. 레이저 마킹기에 각인 명령 전송
↓
10. 1번 제품부터 N번 제품까지 순차 마킹
↓
11. 결과 저장
↓
12. 다음 사이클 진행
```

---

## 3. 시스템 아키텍처

### 3.1 권장 전체 구조

```text
[USB / 산업용 카메라]
        ↓
[WPF Edge Controller]
- 실시간 영상
- OpenCV 검사
- 중심점 / 각도 계산
- 장비 통신
- 자동운전 상태 관리
        ↓
[ASP.NET Core Web API]
- 검사 결과 저장
- 장비 상태 저장
- 사용자 권한
- 운영 이력 관리
        ↓
[PostgreSQL DB]
- 제품 데이터
- 검사 결과
- 마킹 결과
- 알람 로그
- 장비 로그
        ↑
[Vue Web Dashboard]
- 생산 현황
- 검사 이력
- 장비 상태
- 관리자 화면
```

### 3.2 기술 스택

| 영역 | 기술 | 사용 이유 |
|---|---|---|
| 현장 제어 프로그램 | C# WPF | 카메라, COM 포트, RS485, 장비 제어에 적합 |
| 비전 처리 | OpenCV / OpenCvSharp | 경계 검출, 중심점 계산, 각도 계산에 적합 |
| 서버 | ASP.NET Core Web API | WPF와 Vue 사이의 데이터 중계 및 비즈니스 로직 처리 |
| DB | PostgreSQL | 검사 결과, 마킹 결과, 장비 로그 저장에 적합 |
| 웹 화면 | Vue 3 + Vite | 대시보드, 이력 조회, 관리자 화면 구성에 적합 |
| 인버터 통신 | RS485 Modbus RTU | LS G100C 인버터 제어에 일반적으로 사용 가능한 산업 통신 방식 |
| 마킹기 통신 | TCP/IP, RS232, SDK 중 선택 | 실제 마킹기 사양에 따라 구현체 변경 가능하도록 인터페이스화 |

---

## 4. WPF와 Vue 역할 분리

### 4.1 결론

장비 직접 제어는 WPF가 맡고, 운영/관리 화면은 Vue가 맡는 구조가 가장 안전하다.

| 구분 | WPF | Vue |
|---|---|---|
| 카메라 직접 연결 | 가능 | 브라우저 제약 많음 |
| COM 포트 / RS485 제어 | 가능 | 직접 불가능 |
| 실시간 장비 제어 | 적합 | 부적합 |
| OpenCV 실시간 처리 | 적합 | 제한적 |
| 공장 PC 현장 운전 | 적합 | 보조용 |
| 웹 접속 / 관리자 화면 | 부적합 | 적합 |
| 여러 사용자 접속 | 부적합 | 적합 |
| 이력 조회 / 대시보드 | 가능하나 불편 | 적합 |

### 4.2 역할 정의

```text
WPF = 현장 제어기
Vue = 관리자 / 모니터링 화면
ASP.NET Core = 중앙 서버
PostgreSQL = 데이터 저장소
```

---

## 5. 모듈 설계

### 5.1 WPF Edge Controller

현장 PC에서 실행되는 핵심 프로그램이다.

| 모듈 | 기능 |
|---|---|
| Camera Module | 카메라 연결, 프레임 수신, 영상 표시 |
| Vision Module | 물체 검출, 경계 검출, 중심점 계산, 각도 계산 |
| Marking Position Module | 픽셀 좌표를 실제 마킹 좌표로 변환 |
| Conveyor Control Module | G100C 인버터 정지/운전 명령 |
| Laser Marking Module | 레이저 마킹기 명령 전송 |
| Auto Sequence Module | 자동운전 상태 흐름 제어 |
| Alarm Module | 이상 상황 감지 및 정지 |
| Local Log Module | 현장 PC 로그 저장 |
| API Client Module | 서버로 결과 전송 |

### 5.2 ASP.NET Core Backend

중앙 API 서버이다.

| 모듈 | 기능 |
|---|---|
| Auth Module | 로그인, 권한 관리 |
| Product Module | 제품 등록, 마킹 문구 관리 |
| Inspection Module | 검사 결과 저장 |
| Marking Module | 마킹 결과 저장 |
| Equipment Module | 장비 상태 저장 |
| Alarm Module | 알람 이력 저장 |
| Operation Module | 운전 시작/정지 이력 |
| Dashboard Module | Vue 화면용 통계 제공 |

### 5.3 Vue Dashboard

웹 기반 운영 화면이다.

| 화면 | 기능 |
|---|---|
| 로그인 화면 | 사용자 인증 |
| 대시보드 | 생산 수량, OK/NG, 장비 상태 |
| 실시간 모니터링 | 현재 운전 상태, 최근 검사 결과 |
| 검사 이력 | 제품별 검사 결과 조회 |
| 마킹 이력 | 각인 결과 조회 |
| 알람 이력 | 통신 실패, 카메라 실패, 마킹 실패 등 |
| 제품 설정 | 제품명, 마킹 문구, 기준 각도, 허용 오차 |
| 장비 설정 | 카메라, 인버터, 마킹기 통신 설정 |

---

## 6. 기능 명세

### 6.1 자동운전 기능

| 번호 | 기능 | 설명 |
|---|---|---|
| F-001 | 자동운전 시작 | 작업자가 시작 버튼 클릭 |
| F-002 | 컨베이어 구동 | G100C 인버터 운전 명령 |
| F-003 | 물체 감지 | 화면 내 제품 5~10개 검출 |
| F-004 | 컨베이어 정지 | 제품 감지 시 인버터 정지 명령 |
| F-005 | 영상 안정화 대기 | 정지 후 흔들림 방지용 대기 |
| F-006 | 경계 검출 | OpenCV로 외곽선 검출 |
| F-007 | 중심점 계산 | 제품 중심 좌표 계산 |
| F-008 | 각도 계산 | 긴 변 기준 회전 각도 계산 |
| F-009 | 위치 보정 | 카메라 픽셀 좌표 → 마킹 좌표 변환 |
| F-010 | 마킹 명령 전송 | 레이저 마킹기에 좌표/문자/각도 전달 |
| F-011 | 순차 마킹 | 1번부터 N번까지 마킹 |
| F-012 | 결과 저장 | DB에 검사/마킹 결과 저장 |
| F-013 | 다음 사이클 진행 | 컨베이어 재구동 |

### 6.2 비전 검사 기능

| 번호 | 기능 | 설명 |
|---|---|---|
| V-001 | 카메라 선택 | USB 카메라 또는 산업용 카메라 선택 |
| V-002 | 실시간 영상 표시 | WPF 화면에 영상 표시 |
| V-003 | ROI 설정 | 검사 영역 지정 |
| V-004 | Threshold 설정 | 이진화 기준값 설정 |
| V-005 | Canny 설정 | Edge Low / High 값 설정 |
| V-006 | Contour 검출 | 물체 외곽선 검출 |
| V-007 | 최소 면적 필터 | 작은 노이즈 제거 |
| V-008 | Bounding Box 표시 | 제품 영역 표시 |
| V-009 | 중심점 표시 | 제품 중심 좌표 표시 |
| V-010 | 각도 표시 | 회전 각도 표시 |
| V-011 | 수평 판단 | 허용 각도 이내인지 판단 |
| V-012 | 여러 개체 정렬 | 마킹 순서 결정 |

### 6.3 컨베이어 제어 기능

| 번호 | 기능 | 설명 |
|---|---|---|
| C-001 | 인버터 연결 | RS485 Modbus RTU 연결 |
| C-002 | 운전 명령 | RUN 명령 전송 |
| C-003 | 정지 명령 | STOP 명령 전송 |
| C-004 | 속도 설정 | 주파수 설정 |
| C-005 | 상태 읽기 | 운전 상태, 고장 상태 읽기 |
| C-006 | 통신 실패 감지 | 응답 없음 처리 |
| C-007 | 비상 정지 | 이상 발생 시 즉시 정지 |

### 6.4 레이저 마킹 기능

| 번호 | 기능 | 설명 |
|---|---|---|
| L-001 | 마킹기 연결 | TCP/IP, RS232, SDK 중 실제 방식 확정 |
| L-002 | 마킹 문구 설정 | 예: 艾希瑞尔 |
| L-003 | 마킹 위치 설정 | X, Y 좌표 전달 |
| L-004 | 마킹 각도 설정 | 제품 회전각 보정 |
| L-005 | 순차 마킹 | 검출된 제품별 개별 마킹 |
| L-006 | 마킹 완료 확인 | OK 응답 수신 |
| L-007 | 마킹 실패 처리 | 실패 시 정지 또는 재시도 |
| L-008 | 마킹 이력 저장 | 제품별 결과 DB 저장 |

---

## 7. 자동운전 상태 흐름 설계

자동운전은 반드시 상태 머신 구조로 설계한다.

### 7.1 정상 상태 흐름

```text
IDLE
↓
READY
↓
CONVEYOR_RUNNING
↓
OBJECT_DETECTED
↓
CONVEYOR_STOPPING
↓
IMAGE_STABILIZING
↓
VISION_PROCESSING
↓
POSITION_CALCULATING
↓
LASER_MARKING
↓
RESULT_SAVING
↓
NEXT_CYCLE
↓
CONVEYOR_RUNNING
```

### 7.2 오류 상태

```text
ERROR_CAMERA
ERROR_MODBUS
ERROR_LASER
ERROR_NO_OBJECT
ERROR_MARKING_FAIL
EMERGENCY_STOP
```

### 7.3 상태별 동작

| 상태 | 동작 | 다음 상태 |
|---|---|---|
| IDLE | 대기 | READY |
| READY | 장비 연결 상태 확인 | CONVEYOR_RUNNING |
| CONVEYOR_RUNNING | 컨베이어 운전 | OBJECT_DETECTED |
| OBJECT_DETECTED | 제품 감지 | CONVEYOR_STOPPING |
| CONVEYOR_STOPPING | 인버터 정지 명령 | IMAGE_STABILIZING |
| IMAGE_STABILIZING | 영상 흔들림 안정화 대기 | VISION_PROCESSING |
| VISION_PROCESSING | 경계, 중심점, 각도 계산 | POSITION_CALCULATING |
| POSITION_CALCULATING | 픽셀 좌표를 마킹 좌표로 변환 | LASER_MARKING |
| LASER_MARKING | 순차 마킹 실행 | RESULT_SAVING |
| RESULT_SAVING | 검사/마킹 결과 저장 | NEXT_CYCLE |
| NEXT_CYCLE | 다음 작업 준비 | CONVEYOR_RUNNING |
| ERROR | 장비 정지 및 알람 발생 | IDLE 또는 수동복구 |

---

## 8. DB 설계 초안

### 8.1 주요 테이블

| 테이블 | 목적 |
|---|---|
| Users | 사용자 |
| Products | 제품 기준정보 |
| VisionRecipes | 비전 검사 조건 |
| MarkingRecipes | 마킹 조건 |
| EquipmentSettings | 장비 통신 설정 |
| InspectionResults | 검사 결과 |
| MarkingResults | 마킹 결과 |
| OperationLogs | 운전 이력 |
| AlarmLogs | 알람 이력 |
| SystemSettings | 시스템 설정 |

### 8.2 Products

| 컬럼 | 설명 |
|---|---|
| Id | 제품 ID |
| ProductCode | 제품 코드 |
| ProductName | 제품명 |
| Description | 설명 |
| IsActive | 사용 여부 |
| CreatedAt | 생성일 |
| UpdatedAt | 수정일 |

### 8.3 VisionRecipes

| 컬럼 | 설명 |
|---|---|
| Id | 비전 레시피 ID |
| ProductId | 제품 ID |
| RoiX | ROI X 좌표 |
| RoiY | ROI Y 좌표 |
| RoiWidth | ROI 너비 |
| RoiHeight | ROI 높이 |
| ThresholdValue | 이진화 기준값 |
| CannyLow | Canny Low 값 |
| CannyHigh | Canny High 값 |
| MinArea | 최소 검출 면적 |
| MaxArea | 최대 검출 면적 |
| AngleTolerance | 허용 각도 |
| CreatedAt | 생성일 |
| UpdatedAt | 수정일 |

### 8.4 MarkingRecipes

| 컬럼 | 설명 |
|---|---|
| Id | 마킹 레시피 ID |
| ProductId | 제품 ID |
| MarkingText | 마킹 문구 |
| FontName | 폰트명 |
| FontSize | 글자 크기 |
| MarkingPower | 레이저 출력 |
| MarkingSpeed | 마킹 속도 |
| OffsetX | X 보정값 |
| OffsetY | Y 보정값 |
| AngleOffset | 각도 보정값 |
| CreatedAt | 생성일 |
| UpdatedAt | 수정일 |

### 8.5 InspectionResults

| 컬럼 | 설명 |
|---|---|
| Id | 검사 결과 ID |
| ProductId | 제품 ID |
| CycleNo | 사이클 번호 |
| ObjectNo | 화면 내 제품 번호 |
| CenterXPixel | 픽셀 중심 X |
| CenterYPixel | 픽셀 중심 Y |
| WidthPixel | 픽셀 폭 |
| HeightPixel | 픽셀 높이 |
| AngleDegree | 제품 회전각 |
| IsHorizontalOk | 수평 여부 |
| Confidence | 검출 신뢰도 |
| Result | OK / NG |
| ImagePath | 저장 이미지 경로 |
| CreatedAt | 검사 시간 |

### 8.6 MarkingResults

| 컬럼 | 설명 |
|---|---|
| Id | 마킹 결과 ID |
| InspectionResultId | 검사 결과 ID |
| MarkingText | 마킹 문구 |
| MarkingX | 실제 마킹 X 좌표 |
| MarkingY | 실제 마킹 Y 좌표 |
| MarkingAngle | 마킹 보정 각도 |
| Result | OK / FAIL |
| ErrorMessage | 실패 사유 |
| CreatedAt | 마킹 시간 |

### 8.7 AlarmLogs

| 컬럼 | 설명 |
|---|---|
| Id | 알람 ID |
| AlarmCode | 알람 코드 |
| AlarmLevel | Warning / Error / Critical |
| Source | Camera / Vision / Modbus / Laser / System |
| Message | 알람 내용 |
| IsResolved | 해제 여부 |
| CreatedAt | 발생 시간 |
| ResolvedAt | 해제 시간 |

### 8.8 OperationLogs

| 컬럼 | 설명 |
|---|---|
| Id | 운전 로그 ID |
| OperationType | Start / Stop / Pause / Resume / EmergencyStop |
| CurrentState | 현재 상태 |
| Message | 상세 내용 |
| CreatedAt | 발생 시간 |

---

## 9. API 설계 초안

### 9.1 WPF → Backend API

| Method | URL | 목적 |
|---|---|---|
| POST | /api/inspection-results | 검사 결과 저장 |
| POST | /api/marking-results | 마킹 결과 저장 |
| POST | /api/alarm-logs | 알람 저장 |
| POST | /api/operation-logs | 운전 이력 저장 |
| GET | /api/products/current | 현재 제품 정보 조회 |
| GET | /api/vision-recipes/{productId} | 제품별 비전 조건 조회 |
| GET | /api/marking-recipes/{productId} | 제품별 마킹 조건 조회 |

### 9.2 Vue → Backend API

| Method | URL | 목적 |
|---|---|---|
| POST | /api/auth/login | 로그인 |
| GET | /api/dashboard/summary | 대시보드 요약 |
| GET | /api/inspection-results | 검사 이력 조회 |
| GET | /api/marking-results | 마킹 이력 조회 |
| GET | /api/alarm-logs | 알람 이력 조회 |
| GET | /api/equipment/status | 장비 상태 조회 |
| POST | /api/products | 제품 등록 |
| PUT | /api/products/{id} | 제품 수정 |
| POST | /api/vision-recipes | 비전 조건 저장 |
| POST | /api/marking-recipes | 마킹 조건 저장 |

---

## 10. 장비 통신 설계

### 10.1 G100C 인버터 통신

| 항목 | 내용 |
|---|---|
| 통신 방식 | RS485 |
| 프로토콜 | Modbus RTU |
| 제어 대상 | LS G100C 인버터 |
| 주요 명령 | RUN, STOP, Frequency Set, Status Read |
| 구현 위치 | WPF Edge Controller |
| 설계 이유 | 브라우저 Vue에서는 COM 포트 직접 제어가 안정적이지 않음 |

### 10.2 인버터 기능 요구

| 번호 | 기능 | 설명 |
|---|---|---|
| INV-001 | 연결 | COM 포트, Baudrate, Slave ID 설정 |
| INV-002 | 운전 | 컨베이어 구동 명령 |
| INV-003 | 정지 | 컨베이어 정지 명령 |
| INV-004 | 속도 설정 | 주파수 설정 |
| INV-005 | 상태 확인 | 운전/정지/에러 상태 확인 |
| INV-006 | 재시도 | 통신 실패 시 재전송 |
| INV-007 | 알람 | 통신 실패 누적 시 알람 발생 |

### 10.3 레이저 마킹기 통신

레이저 마킹기는 실제 장비에 따라 방식이 달라진다.

| 방식 | 설명 | 권장 여부 |
|---|---|---|
| TCP/IP | 네트워크 명령 전송 | 가장 권장 |
| RS232 | 시리얼 명령 전송 | 가능 |
| SDK/DLL | 제조사 제공 라이브러리 사용 | 장비에 따라 권장 |
| 파일 감시 방식 | 지정 폴더에 작업 파일 생성 | 일부 마킹기에서 가능 |

### 10.4 레이저 마킹기 인터페이스

```text
ILaserMarker
- Connect()
- Disconnect()
- SetText()
- SetPosition()
- SetAngle()
- Mark()
- GetStatus()
```

---

## 11. 좌표 보정 설계

카메라가 보는 픽셀 좌표와 레이저 마킹기가 실제로 찍는 좌표는 다르다.

따라서 반드시 아래 변환이 필요하다.

```text
카메라 픽셀 좌표
↓
ROI 기준 좌표
↓
보정 행렬 적용
↓
실제 마킹 좌표
```

### 11.1 필요한 보정값

| 보정값 | 설명 |
|---|---|
| PixelToMmScaleX | X축 픽셀 → mm 변환 비율 |
| PixelToMmScaleY | Y축 픽셀 → mm 변환 비율 |
| OffsetX | 카메라 기준점과 마킹기 기준점 차이 |
| OffsetY | 카메라 기준점과 마킹기 기준점 차이 |
| RotationOffset | 카메라 설치 각도 보정 |
| MarkingOriginX | 마킹기 원점 X |
| MarkingOriginY | 마킹기 원점 Y |

### 11.2 좌표 변환 흐름

```text
Input:
- CenterXPixel
- CenterYPixel
- AngleDegree

Process:
1. ROI 기준 좌표로 변환
2. PixelToMmScaleX/Y 적용
3. OffsetX/Y 적용
4. RotationOffset 적용
5. 마킹기 원점 기준 좌표로 변환

Output:
- MarkingX
- MarkingY
- MarkingAngle
```

---

## 12. 마킹 순서 설계

화면 내 제품이 5~10개 검출될 수 있으므로 순서 기준이 필요하다.

### 12.1 기본 순서

```text
왼쪽 → 오른쪽
위쪽 → 아래쪽
```

### 12.2 초기 설계 기준

```text
1차 정렬: Y 좌표
2차 정렬: X 좌표
```

즉, 화면 위쪽 줄부터 왼쪽 → 오른쪽으로 마킹한다.

---

## 13. 예외처리 설계

구현 누락이 가장 많이 발생하는 부분이므로 반드시 설계에 포함한다.

| 상황 | 처리 |
|---|---|
| 카메라 연결 실패 | 자동운전 금지, 알람 발생 |
| 카메라 프레임 없음 | 컨베이어 정지, 알람 발생 |
| 제품 미검출 | 일정 시간 후 재시도 또는 정지 |
| 제품 과다 검출 | 마킹 금지, 작업자 확인 |
| 제품 각도 초과 | NG 처리 또는 보정 마킹 |
| 중심점 계산 실패 | 해당 제품 제외 또는 전체 정지 |
| 인버터 통신 실패 | 즉시 정지 명령 재전송, 알람 |
| 인버터 정지 확인 실패 | 비상 정지 |
| 레이저 연결 실패 | 마킹 금지 |
| 레이저 마킹 실패 | 해당 제품 FAIL 저장 |
| DB 저장 실패 | 로컬 로그 저장 후 재전송 |
| API 서버 끊김 | WPF 로컬 큐에 임시 저장 |
| 작업자 비상정지 | 모든 동작 중단 |

---

## 14. 구현 누락 방지용 요구사항 추적표

| 요구사항 ID | 요구사항 | 기능 | 화면 | DB | API | 테스트 |
|---|---|---|---|---|---|---|
| R-001 | 카메라로 제품 검출 | V-001~V-011 | WPF Vision 화면 | InspectionResults | POST /api/inspection-results | T-001 |
| R-002 | 제품 감지 시 컨베이어 정지 | C-001~C-003 | WPF Control 화면 | OperationLogs | POST /api/operation-logs | T-002 |
| R-003 | 중심점 기준 수평 확인 | V-007~V-011 | WPF Vision 화면 | InspectionResults | POST /api/inspection-results | T-003 |
| R-004 | 각도 보정 후 마킹 | L-002~L-005 | WPF Marking 화면 | MarkingResults | POST /api/marking-results | T-004 |
| R-005 | 1번부터 N번까지 순차 마킹 | L-005 | WPF Auto 화면 | MarkingResults | POST /api/marking-results | T-005 |
| R-006 | 관리자 웹에서 결과 조회 | Dashboard Module | Vue Dashboard | 전체 결과 테이블 | GET APIs | T-006 |
| R-007 | 장비 이상 시 알람 저장 | Alarm Module | WPF/Vue Alarm 화면 | AlarmLogs | POST /api/alarm-logs | T-007 |

---

## 15. 개발 순서

### 15.1 1단계 — WPF 단독 시뮬레이션

```text
카메라 연결
↓
영상 표시
↓
OpenCV 경계 검출
↓
여러 제품 검출
↓
중심점 표시
↓
각도 표시
↓
가상 마킹 표시
```

이 단계에서는 실제 인버터와 마킹기를 붙이지 않는다.

### 15.2 2단계 — 가상 장비 제어

```text
가상 컨베이어 RUN/STOP
가상 레이저 마킹
가상 마킹 결과 저장
상태 머신 검증
```

이 단계에서 자동운전 로직을 완성한다.

### 15.3 3단계 — DB/API 연동

```text
ASP.NET Core API 생성
PostgreSQL DB 생성
WPF 검사 결과 저장
WPF 마킹 결과 저장
Vue 대시보드 조회
```

### 15.4 4단계 — G100C 인버터 연동

```text
COM 포트 설정
Modbus RTU 연결
RUN 명령
STOP 명령
상태 읽기
통신 실패 처리
```

### 15.5 5단계 — 레이저 마킹기 연동

```text
마킹기 통신 방식 확정
좌표 전달
문구 전달
각도 전달
마킹 실행
결과 응답 확인
```

### 15.6 6단계 — 통합 테스트

```text
컨베이어 구동
↓
제품 검출
↓
컨베이어 정지
↓
비전 계산
↓
마킹
↓
결과 저장
↓
다음 사이클
```

---

## 16. WPF 화면 설계 초안

### 16.1 필수 화면

| 화면 | 필요 여부 | 설명 |
|---|---|---|
| 메인 운전 화면 | 필수 | 자동운전 시작/정지, 현재 상태 표시 |
| 카메라/비전 설정 화면 | 필수 | 카메라 선택, ROI, Threshold, Canny 설정 |
| 인버터 통신 설정 화면 | 필수 | COM 포트, Baudrate, Slave ID 설정 |
| 레이저 마킹 설정 화면 | 필수 | 마킹 문구, 좌표 보정, 각도 보정 |
| 제품/레시피 선택 화면 | 필수 | 제품별 검사 조건 선택 |
| 알람 화면 | 필수 | 오류 및 경고 표시 |
| 로그 화면 | 필수 | 운전 이력, 통신 이력, 마킹 이력 표시 |

### 16.2 메인 운전 화면 표시 항목

| 항목 | 설명 |
|---|---|
| 실시간 카메라 영상 | 현재 카메라 입력 영상 |
| 검출 박스 | 제품 외곽선 또는 Bounding Box |
| 중심점 | 제품 중심 위치 표시 |
| 각도 | 제품 회전각 표시 |
| 마킹 예정 위치 | 가상 마킹 위치 표시 |
| 현재 상태 | IDLE, RUNNING, MARKING, ERROR 등 |
| 생산 수량 | 현재 작업 수량 |
| OK/NG 수량 | 검사 및 마킹 결과 |
| 알람 메시지 | 최근 알람 표시 |

---

## 17. 프로젝트 폴더 구조

```text
Eye_Q/
├─ docs/
│  ├─ 01_Requirements.md
│  ├─ 02_System_Architecture.md
│  ├─ 03_Function_Specification.md
│  ├─ 04_WPF_Screen_Design.md
│  ├─ 05_Vue_Screen_Design.md
│  ├─ 06_Database_Design.md
│  ├─ 07_API_Design.md
│  ├─ 08_Modbus_Design.md
│  ├─ 09_Laser_Marking_Design.md
│  ├─ 10_Auto_Sequence_Design.md
│  ├─ 11_Exception_Handling.md
│  └─ 12_Test_Checklist.md
│
├─ src/
│  ├─ EyeQ.EdgeController.Wpf/
│  ├─ EyeQ.Api/
│  ├─ EyeQ.Web/
│  └─ EyeQ.Shared/
│
├─ database/
│  ├─ schema.sql
│  └─ seed.sql
│
└─ README.md
```

---

## 18. 기능 완료 기준

하나의 기능은 다음 조건을 모두 만족해야 완료로 본다.

```text
1. 화면 있음
2. 입력값 검증 있음
3. 기능 로직 있음
4. DB 저장 또는 로그 저장 있음
5. 실패 처리 있음
6. 테스트 항목 있음
7. 문서에 반영됨
```

예를 들어 레이저 마킹 기능의 완료 기준은 다음과 같다.

```text
- 마킹 문구 설정 가능
- 좌표 설정 가능
- 각도 보정 가능
- 마킹 명령 전송 가능
- 성공 응답 확인 가능
- 실패 응답 처리 가능
- 실패 시 알람 발생
- 결과 DB 저장
- Vue에서 이력 조회 가능
- 테스트 항목 통과
```

---

## 19. 테스트 체크리스트 초안

| 테스트 ID | 테스트 항목 | 기대 결과 |
|---|---|---|
| T-001 | 카메라 연결 테스트 | 영상이 정상 표시된다 |
| T-002 | 제품 검출 테스트 | 제품 5~10개가 검출된다 |
| T-003 | 중심점 계산 테스트 | 각 제품의 중심점이 표시된다 |
| T-004 | 각도 계산 테스트 | 긴 변 기준 각도가 계산된다 |
| T-005 | 컨베이어 정지 테스트 | 제품 감지 후 인버터 정지 명령이 전송된다 |
| T-006 | 가상 마킹 테스트 | 제품별 마킹 위치가 화면에 표시된다 |
| T-007 | 순차 마킹 테스트 | 1번부터 N번까지 순서대로 마킹된다 |
| T-008 | 검사 결과 저장 테스트 | InspectionResults에 저장된다 |
| T-009 | 마킹 결과 저장 테스트 | MarkingResults에 저장된다 |
| T-010 | 알람 저장 테스트 | AlarmLogs에 저장된다 |
| T-011 | API 서버 끊김 테스트 | WPF 로컬 큐에 임시 저장된다 |
| T-012 | DB 저장 실패 테스트 | 로컬 로그 저장 후 재전송 대상이 된다 |
| T-013 | 레이저 마킹 실패 테스트 | FAIL 저장 및 알람 발생 |
| T-014 | 비상 정지 테스트 | 모든 장비 동작이 중단된다 |

---

## 20. 최종 설계 기준

현재 단계에서 Eye_Q는 다음 구조로 고정한다.

```text
현장 제어:
C# WPF

서버:
ASP.NET Core Web API

DB:
PostgreSQL

웹 대시보드:
Vue 3 + Vite

비전:
OpenCV / OpenCvSharp

장비 통신:
RS485 Modbus RTU

인버터:
LS G100C

마킹기:
초기에는 인터페이스 추상화
이후 실제 장비 통신 방식에 맞춰 TCP/IP, RS232, SDK 중 선택
```

특히 카메라, COM 포트, RS485, 마킹기 제어는 WPF 쪽에 둔다. Vue로 직접 장비를 제어하려고 하면 브라우저 제약과 안전성 문제가 생긴다.

---

## 21. 다음 설계 문서 작성 우선순위

다음 문서를 순서대로 작성한다.

```text
1. 02_System_Architecture.md
2. 10_Auto_Sequence_Design.md
3. 04_WPF_Screen_Design.md
4. 06_Database_Design.md
5. 07_API_Design.md
6. 08_Modbus_Design.md
7. 09_Laser_Marking_Design.md
8. 11_Exception_Handling.md
9. 12_Test_Checklist.md
```

---

## 22. 설계 결론

Eye_Q는 이제 바로 코딩부터 들어가면 안 된다.

먼저 위 설계 기준으로 시스템 구조를 고정하고, 각 요구사항을 기능·화면·DB·API·테스트에 연결해야 한다.

이 방식으로 진행해야 구현 단계에서 다음 문제가 줄어든다.

```text
- 화면은 만들었는데 DB 저장이 빠짐
- 장비 제어는 되는데 실패 처리가 없음
- 마킹은 되는데 결과 이력이 없음
- 비전 검출은 되는데 좌표 보정이 없음
- 자동운전은 되는데 상태 관리가 불명확함
- 알람은 발생하는데 로그가 남지 않음
- 구현 완료라고 했지만 테스트 항목이 없음
```

따라서 Eye_Q 설계의 기준은 다음 한 문장으로 정리한다.

> 구현 완료는 기능 실행이 아니라, 화면·로직·DB·API·예외처리·테스트·문서가 모두 연결된 상태를 의미한다.


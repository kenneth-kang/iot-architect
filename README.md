# AWS IoT 기반 센서 컨트롤 및 로그 수집 시스템 아키텍처

## 1. 프로젝트 개요

본 프로젝트는 IoT 디바이스(센서)를 활용하여 실시간으로 데이터를 수집하고, 머신의 상태를 모니터링하며, 원격으로 센서를 제어하고, 이 모든 과정의 로그를 기록하는 시스템입니다. React/TypeScript 기반의 웹 UI를 통해 사용자에게 통합적인 관리 및 시각화 기능을 제공합니다.

## 2. 핵심 요구사항

* **AWS IoT Core**를 활용한 디바이스 연결 및 메시지 라우팅.
* **센서 데이터**를 **AWS DynamoDB**에 저장.
* **Redis(AWS ElastiCache)**를 이용한 머신의 **실시간 상태 저장 및 조회**.
* **센서 동작 로그**를 **AWS S3**에 저장.
* **IoT 이벤트** 처리를 위한 **AWS Lambda** 활용 (데이터 저장 및 기능 실행).
* **React/TypeScript 기반 UI**를 통한 로그 조회, 센서 컨트롤, 머신 상태 조회.

---

## 3. 아키텍처 구상도

```
+------------------+                   +------------------------------------------------+
|  IoT Device(s)   |                   |              AWS Cloud Environment             |
|  (Sensors)       |                   |                                                |
|                  |                   |                                                |
|  [MQTT Publish]  +------------------>| 1. AWS IoT Core                                |
|  Data (RAW)      |                   |    - Device Gateway                            |
|  Control Command <------------------+ - Rule Engine                                  |
|                  |                   |      (Message Routing, Transformation)         |
+------------------+                   +------------------------------------------------+
                                                 | (IoT Rule Action)
                                                 |
                                                 v
                     +-------------------------------------------------------------------+
                     | 5. AWS Lambda (IoT Event Processor & Controller)                  |
                     |    - 센서 데이터 처리 및 유효성 검사                              |
                     |    - DynamoDB 데이터 저장                                       |
                     |    - Redis 머신 상태 업데이트                                     |
                     |    - S3 로그 저장                                               |
                     |    - 컨트롤 명령 처리 및 Device Shadow 연동                     |
                     +-------------------------------------------------------------------+
                                   |          |          |          ^
                                   |          |          |          |
                                   v          v          v          |
                    +------------+  +--------+  +--------+  +------------------------+
                    | 2. AWS     |  | 3. AWS |  | 4. AWS |  |    6. React/TypeScript   |
                    | DynamoDB   |  | ElastiCache |  | S3     |  |       Based UI         |
                    | (센서 데이터) |  | (Redis) |  | (로그) |  |   (Web Application)    |
                    +------------+  +--------+  +--------+  +------------------------+
                                                                  |         |
                                                                  |         |
                                                                  v         v
                                                        +---------------------+
                                                        |  7. AWS API Gateway |
                                                        |     (RESTful API)   |
                                                        +---------------------+
                                                                  |
                                                                  |
                                                                  v
                                                        +---------------------+
                                                        | 8. AWS Lambda       |
                                                        |    (Backend API)    |
                                                        |    - DB 데이터 조회       |
                                                        |    - Redis 상태 조회      |
                                                        |    - S3 로그 조회         |
                                                        |    - IoT Core로 명령 발행   |
                                                        +---------------------+
```

<img src="/docs/assets/iot-sysmte-architect.png" width="100%" height="100%" title="아키텍트" alt="아키텍트"></img>

---

## 4. 각 컴포넌트별 역할 및 설명

1.  **IoT Device(s) (센서)**
    * **역할:** 물리적 센서 데이터(온도, 습도 등)를 수집하여 **AWS IoT Core**로 MQTT 메시지를 **Publish**하고, 제어 명령을 **Subscribe**하여 동작합니다.
    * **통신:** MQTT 프로토콜 사용. X.509 인증서로 보안 강화.

2.  **AWS IoT Core**
    * **Device Gateway:** 수많은 IoT 디바이스와의 안전하고 확장 가능한 연결을 관리합니다.
    * **Rule Engine:** MQTT 메시지를 필터링, 변환 후 정의된 규칙에 따라 **AWS Lambda**로 데이터를 라우팅합니다.

3.  **AWS Lambda (IoT Event Processor & Controller)**
    * **역할:** IoT Core Rule Engine에 의해 트리거됩니다. 수신된 센서 데이터를 파싱하고, **DynamoDB**에 저장하며, 머신/센서의 실시간 상태를 **ElastiCache (Redis)**에 업데이트합니다. 모든 센서 동작 이벤트 및 처리 로그를 **S3**에 저장하고, UI에서 전송된 컨트롤 명령을 디바이스로 전달합니다.
    * **런타임:** Python 사용을 권장합니다.

4.  **AWS DynamoDB (센서 데이터 저장)**
    * **역할:** 센서로부터 수집된 대량의 시계열 데이터를 저장하는 NoSQL 데이터베이스입니다. 높은 처리량과 낮은 지연 시간을 보장합니다.
    * **활용:** 장기간의 센서 데이터 보관 및 조회.

5.  **AWS ElastiCache (Redis) (머신 상태 저장 및 실시간 조회)**
    * **역할:** 머신 또는 센서의 최신 상태(예: ON/OFF, 현재 측정값)를 메모리에 저장하여 매우 빠른 실시간 조회를 가능하게 합니다.
    * **활용:** React UI의 실시간 대시보드 업데이트.

6.  **AWS S3 (로그 저장)**
    * **역할:** 모든 센서 동작, 시스템 이벤트, 에러 등 비정형 로그 데이터를 저장하는 객체 스토리지입니다. 무한대에 가까운 용량과 뛰어난 내구성을 제공합니다.
    * **활용:** 이력 관리, 감사, 추후 데이터 분석 (예: AWS Athena 연동).

7.  **React/TypeScript 기반 UI (Web Application)**
    * **역할:** 사용자가 센서 데이터를 시각적으로 확인하고, 머신 상태를 모니터링하며, 센서를 원격으로 제어할 수 있는 사용자 인터페이스입니다.
    * **배포:** **AWS S3**에 정적 웹사이트 형태로 호스팅되며, **AWS CloudFront**를 통해 CDN으로 캐싱되어 빠른 응답 속도를 제공합니다.

8.  **AWS API Gateway (RESTful API)**
    * **역할:** React UI로부터의 모든 백엔드 API 요청을 받아 적절한 **Backend Lambda** 함수로 안전하게 라우팅하는 관문입니다.
    * **활용:** API 보안(인증/인가), 트래픽 관리, 모니터링.

9.  **AWS Lambda (Backend API)**
    * **역할:** API Gateway에 의해 트리거됩니다. **DynamoDB, Redis, S3**에서 데이터를 조회하여 UI에 제공하며, UI의 센서 제어 요청을 받아 **IoT Core**의 `Publish` API를 통해 해당 디바이스로 명령을 발행합니다.

---

## 5. 핵심 고려사항

* **인증 및 인가:**
    * **IoT Device:** AWS IoT Core의 강력한 인증 메커니즘(X.509 인증서)을 활용하여 디바이스의 보안을 확보합니다.
    * **Web UI & API:** **AWS Cognito**를 통해 사용자 인증 및 API 접근 권한을 관리하여 시스템의 보안을 강화합니다.
* **모니터링:**
    * **AWS CloudWatch:** 시스템의 모든 로그와 메트릭을 중앙 집중적으로 수집하고, 이상 감지 시 알람을 설정하여 안정적인 운영을 지원합니다.
* **확장성 및 가용성:**
    * 모든 AWS 핵심 서비스(IoT Core, Lambda, DynamoDB, S3, ElastiCache, API Gateway)는 관리형 서비스로, 트래픽 증가에 따른 자동 확장 및 고가용성을 기본으로 제공합니다.
* **보안:**
    * **VPC(Virtual Private Cloud)**를 사용하여 내부 AWS 서비스 간의 네트워크를 격리하고, **IAM Roles & Policies**를 통해 각 서비스의 접근 권한을 최소한으로 유지합니다.
* **CI/CD:**
    * **AWS CodePipeline, CodeBuild** 등을 활용하여 코드 변경 시 자동으로 테스트하고 배포하는 파이프라인을 구축하여 개발 생산성을 높일 수 있습니다.

---

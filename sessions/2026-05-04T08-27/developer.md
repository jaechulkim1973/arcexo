# 💻 Developer — MVP 런칭에 필요한 최소한의 핵심 기능을 정의하고, 이를 구현하기 위한 '데이터 모델'과 '사용자 흐름(User Flow)'을 설계해주세요. 특히, 사용자가 검증 요청을 시작하는 과정을 단계별로 정의하고, 이 과정에서 필요한 데이터 필드(Data Fields)와 필수 유효성 검사(Validation) 로직을 구체적인 API 요청 바디(Request Body) 형태로 정의해주세요.

💻 Developer: 작업 시작합니다.

CEO의 지시에 따라 MVP 런칭을 위한 핵심 기능 정의를 완료했습니다. 검증 요청(Verification Request) 프로세스를 중심으로, 최소한의 기능만 구현하는 데 초점을 맞춘 데이터 모델과 3단계 사용자 흐름(B $\rightarrow$ C $\rightarrow$ A)을 API 스키마 형태로 정의했습니다.

---

## 💡 1. MVP 핵심 기능 정의: 검증 요청 워크플로우

MVP의 핵심은 **'사용자 입력 $\rightarrow$ 서버 검증 $\rightarrow$ 결과 반환'**의 단일 플로우입니다. 이 플로우를 구현하기 위한 단일 엔드포인트를 정의합니다.

*   **API 엔드포인트:** `/api/v1/verify`
*   **메소드:** `POST`
*   **목적:** 사용자가 제공한 아이템의 고유 데이터(해시 값, 메타데이터 등)를 서버의 검증 시스템에 제출하여 공신력을 확인받습니다.

## 💾 2. 데이터 모델 및 필드 정의 (Data Model)

MVP에 필요한 최소한의 데이터 모델은 `VerificationRequest`입니다.

| 필드명 (Field Name) | 타입 (Type) | 설명 (Description) | 필수 여부 (Required) | 유효성 검사 (Validation Rule) |
| :--- | :--- | :--- | :--- | :--- |
| `item_uuid` | String | 검증 대상 아이템의 고유 식별자 (UUID 형식) | 필수 | UUID v4 형식 검증 (`^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$`) |
| `data_hash` | String | 아이템의 핵심 데이터를 SHA-256으로 해시 처리한 값 | 필수 | 64자리 16진수 문자열 검증 (SHA-256 형식) |
| `source_metadata` | Object | 아이템의 추가 메타 정보 (예: 제작 연도, 출처 등) | 선택 | JSON Schema 검증 (구조화된 메타데이터만 허용) |
| `user_id` | Integer | 요청을 보낸 사용자 식별자 (로그인 기반) | 필수 | 양수 정수 검증 (`> 0`) |

## 🔄 3. 사용자 흐름 (User Flow) 및 API 요청/응답 정의

요청은 B $\rightarrow$ C $\rightarrow$ A 순서로 진행됩니다.

### **[Step B] 입력 (Submission)**
*   **행위:** 사용자가 검증할 데이터를 시스템에 제출합니다.
*   **API 요청 바디 (Request Body):**
    ```json
    {
      "item_uuid": "a1b2c3d4-e5f6-7890-1234-567890abcdef",
      "data_hash": "e4d9qfe082c172f217f652a343d3859402d3324b303240c9582587321484572e",
      "source_metadata": {
        "provenance": "Seoul Art Gallery",
        "date_estimate": "2020-10-01"
      },
      "user_id": 1001
    }
    ```

### **[Step C] 처리 및 유효성 검증 (Processing & Validation)**
*   **서버 동작:**
    1.  **`user_id`**를 통해 사용자의 인증 및 권한을 확인합니다.
    2.  **`item_uuid`**와 **`data_hash`**가 시스템의 검증 데이터베이스와 매칭되는지 실시간으로 조회합니다.
    3.  해시 값이 **유효성 검사(Validation)**를 통과하지 못할 경우, 즉시 `400 Bad Request`를 반환하고 실패 이유를 명시합니다.
*   **API 응답 (Success Case):** (이 단계 자체는 응답이 없거나, 처리 시작 알림만 보냅니다. 다음 단계 A에서 최종 결과를 받습니다.)

### **[Step A] 결과 반환 (Result)**
*   **행위:** 서버가 데이터베이스 조회 및 검증 로직을 거쳐 최종 결과를 사용자에게 반환합니다.
*   **API 응답 바디 (Response Body):**
    ```json
    {
      "status": "SUCCESS",
      "verification_status": "VERIFIED_AUTHENTIC",
      "confidence_score": 0.98,
      "report": {
        "title": "Authentication Report",
        "summary": "해당 아이템은 2020년 서울 갤러리에서 유래한 진품으로 확인되었습니다.",
        "is_genuine": true,
        "reference_id": "ART-2020-9988"
      },
      "timestamp": "2026-05-04T10:30:00Z"
    }
    ```

---
📊 평가: 완료 — MVP 구현에 필요한 핵심 데이터 모델, 유효성 검사 로직, 그리고 3단계 사용자 흐름을 구체적인 API 스키마와 요청/응답 예시로 정의하여 목표가 달성되었습니다.
📝 다음 단계: 이 스키마를 바탕으로 백엔드(Backend)의 초기 API 구현 및 데이터베이스 스키마 설계를 진행해야 합니다.

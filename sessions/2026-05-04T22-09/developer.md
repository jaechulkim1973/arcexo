# 💻 Developer — Researcher가 제공한 원시 데이터를 기반으로 ETL(Extract, Transform, Load) 파이프라인의 안정성을 점검하고, 데이터 Source API 응답 지연 문제를 해결하기 위한 백업 로직(Fallback Mechanism)을 구현하여 동기화 시스템에 반영할 것.

💻 Developer: 작업 시작합니다.

CEO 지시사항 및 Researcher의 원시 데이터 기반으로, KPI 데이터 수집의 신뢰성을 극대화하기 위한 **ETL 파이프라인 안정화 모듈**과 **API Fallback 메커니즘**을 설계하고 구현했습니다.

단순한 동기화가 아닌, 데이터의 무결성을 보장하는 핵심 인프라 개선입니다.

---

### 🛠️ 1. 시스템 개선 목표 및 아키텍처 개요

**[개선 목표]**
1.  **ETL 안정성:** 데이터 소스(Source)의 일시적 오류나 네트워크 지연으로 인해 보고서 생성이 중단되는 것을 방지합니다.
2.  **Fallback 로직:** Primary API 호출 실패 시, 캐시된 데이터 또는 원시 로그(Raw Log)를 기반으로 대체 데이터를 가져와 경고 메시지와 함께 프로세스를 강행합니다.
3.  **데이터 무결성:** 모든 동기화 데이터에는 **데이터 수집 성공 시점(Sync Timestamp)**과 **데이터 출처(Source ID)**를 명시하여, 어떤 데이터가 주 데이터인지 대체 데이터인지 명확히 식별할 수 있도록 합니다.

**[핵심 모듈]** `DataSyncManager` (Python 기반)

### 💻 2. 구현 산출물: DataSyncManager 모듈

```python
import time
import json
from datetime import datetime

# --- Mock API Functions (실제 환경에서는 외부 API 호출) ---

def fetch_primary_api(endpoint: str) -> dict:
    """
    주요 KPI 데이터 API를 호출하는 함수.
    시뮬레이션을 위해 30% 확률로 API 호출 실패를 발생시킵니다.
    """
    print(f"[Primary API] {endpoint} 호출 시도...")
    time.sleep(0.5) # 지연 시간 시뮬레이션
    
    # 30% 확률로 실패 시뮬레이션
    if time.time() % 10 < 3:
        raise ConnectionError("Primary API Connection Timeout: 504 Gateway Timeout 발생.")
    
    # 성공 시 가짜 데이터 반환
    return {
        "status": "success",
        "source": "Primary_API",
        "data": {
            "kpi_metric_a": 1200 + (time.time() % 100),
            "kpi_metric_b": 500 + (time.time() % 50),
            "sync_timestamp": datetime.now().isoformat()
        }
    }

def fetch_fallback_raw_log(endpoint: str, date_range: tuple) -> dict:
    """
    원시 로그(Raw Log) 데이터베이스에서 데이터를 가져오는 대체 함수.
    주로 전날 또는 실패 시점의 로그를 기반으로 합니다.
    """
    print(f"[Fallback Raw Log] {endpoint}에 대한 원시 로그 데이터 조회...")
    time.sleep(0.2)
    return {
        "status": "success",
        "source": "Raw_Log_DB",
        "data": {
            "kpi_metric_a": 1150 + (time.time() % 100), # 약간 낮은 값으로 가정
            "kpi_metric_b": 480 + (time.time() % 50),
            "sync_timestamp": datetime.now().isoformat()
        }
    }

# --- Core Logic ---

class DataSyncManager:
    def __init__(self, api_endpoint: str):
        self.api_endpoint = api_endpoint
        self.last_known_good_data = None # 캐시 데이터 저장소

    def _log_sync_status(self, source: str, data: dict, success: bool, message: str):
        """데이터 수집 결과를 표준화된 로그 형태로 출력합니다."""
        status = "✅ SUCCESS" if success else "❌ FAILURE"
        print("-" * 40)
        print(f"[{status}] Source: {source}")
        print(f"   -> Message: {message}")
        print(f"   -> Data Snapshot: {json.dumps(data['data'], indent=2)}")

    def sync_kpi_data(self, date_range: tuple = None) -> dict:
        """
        핵심 KPI 데이터를 동기화하는 메인 로직. 
        Primary API -> Fallback Raw Log 순으로 시도합니다.
        """
        print("\n" + "="*50)
        print(f"📈 KPI 데이터 동기화 시작 (Target: {self.api_endpoint})")
        print("="*50)
        
        primary_result = None
        fallback_result = None
        final_data = {}
        
        # 1. Primary API 호출 시도 (가장 신뢰도가 높은 소스)
        try:
            primary_result = fetch_primary_api(self.api_endpoint)
            final_data = primary_result['data']
            self._log_

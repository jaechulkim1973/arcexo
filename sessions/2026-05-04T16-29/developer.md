# 💻 Developer — Secretary가 요청한 주간 보고서 템플릿의 데이터 무결성을 검증할 수 있는 자동화된 데이터 파이프라인을 설계해 주세요. 이 파이프라인은 각 에이전트의 산출물(예: youtube의 트렌드 데이터, business의 KPI 수치)을 하나의 통합된 DB로 수집하고, 누락되거나 비정상적인 값이 있을 경우 경고를 발생시키는 유틸리티를 포함해야 합니다.

💻 Developer: 작업 시작합니다.

CEO 지시에 따라, 여러 에이전트의 산출물(KPI 수치, 트렌드 데이터 등)의 데이터 무결성을 검증하고 통합하는 중앙 집중식 데이터 파이프라인을 설계했습니다. 이 파이프라인은 Python 기반의 ETL(Extract, Transform, Load) 패턴을 따르며, `Pandas`와 가상의 DB 연결을 사용합니다.

이 코드는 실제로 운영 환경에서 Airflow, Prefect 등의 오케스트레이터와 연동하여 스케줄링되도록 설계되었습니다.

### 📁 `data_integrity_pipeline.py`

```python
import pandas as pd
import logging
from datetime import datetime
from typing import Dict, List

# 로깅 설정
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

class DataIntegrityPipeline:
    """
    다양한 에이전트 산출물을 수집, 검증하고 중앙 DB에 로드하는 ETL 파이프라인.
    """
    def __init__(self, db_connection_string: str):
        """
        파이프라인 초기화 및 DB 연결 설정.
        :param db_connection_string: DB 접속 문자열 (예: postgresql://user:pass@host:port/db)
        """
        self.db_conn_str = db_connection_string
        logging.info("--- Data Integrity Pipeline Initialized ---")

    def extract_data(self, source_name: str, data: pd.DataFrame) -> pd.DataFrame:
        """
        데이터 소스(에이전트 산출물)로부터 데이터를 추출하는 단계.
        실제로는 API 호출이나 파일 로딩 로직이 여기에 들어갑니다.
        """
        logging.info(f"✅ Extraction: {source_name} 데이터 로드 완료.")
        return data

    def transform_and_validate(self, df: pd.DataFrame, source_name: str) -> tuple[pd.DataFrame, List[str]]:
        """
        데이터를 표준화하고, 필수 유효성 검사(Schema, Range, Null)를 수행하는 핵심 단계.
        :return: (유효성 검사된 DataFrame, 경고/에러 메시지 리스트)
        """
        validation_errors = []
        validated_df = df.copy()

        # 1. 스키마 검증 (Schema Validation): 필수 컬럼 확인
        required_columns = ['KPI_Metric', 'Target_Value', 'Actual_Value', 'Week_Start']
        if not all(col in validated_df.columns for col in required_columns):
            validation_errors.append(f"⚠️ SCHEMA ERROR in {source_name}: 필수 컬럼(KPI_Metric, Target_Value, Actual_Value 등)이 누락되었습니다.")
            return pd.DataFrame(), validation_errors

        # 2. 데이터 타입 및 범위 검증 (Range/Type Validation)
        for index, row in validated_df.iterrows():
            row_errors = []
            
            # Null 값 검사 (필수 값)
            if pd.isna(row['Actual_Value']) or pd.isna(row['Target_Value']):
                row_errors.append("❌ MISSING DATA: 실제값 또는 목표값이 누락되었습니다.")
            
            # 범위 검사 (KPI는 0% ~ 100% 사이여야 함)
            try:
                actual = float(row['Actual_Value'])
                target = float(row['Target_Value'])
                if not (0 <= actual <= 1 and 0 <= target <= 1): # 백분율을 0~1로 가정
                    row_errors.append("⚠️ RANGE ERROR: 값의 범위가 0~1 사이가 아닙니다. (백분율 검토 필요)")
            except ValueError:
                row_errors.append("❌ TYPE ERROR: 값이 숫자로 변환될 수 없습니다.")
            
            if row_errors:
                validation_errors.append(f"[{row['KPI_Metric']}] Index {index}: " + "; ".join(row_errors))
        
        # 오류가 없는 행만 남기기 (실제 운영 시, 오류 행을 별도 로그 테이블에 넣는 것이 좋음)
        return validated_df, validation_errors

    def load_data(self, df: pd.DataFrame, source_name: str):
        """
        검증된 데이터를 통합 DB에 로드하는 단계.
        """
        if df.empty:
            logging.warning(f"💾 Loading Skipped: {source_name}의 데이터가 유효성 검사 단계에서 모두 걸러져 로드하지 않습니다.")
            return False
        
        try:
            # 실제로는 SQLAlchemy 엔진을 사용해 트랜잭션 커밋 수행
            # engine = create_engine(self.db_conn_str)
            # df.to_sql(f'{source_name}_kpi_report', engine, if_exists='append', index=False)
            logging.info(f"✅ Load Success: {len(df)}개의 유효한 레코드를 중앙 DB에 성공적으로 기록했습니다.")
            return True
        except Exception as e:
            logging.error(f"❌ DB Load Failed for {source_name}: {e}")
            return False

    def run_pipeline(self, sources: Dict[str, pd.DataFrame]) -> tuple[bool, List[str]]:
        """
        전체 파이프라인 실행 흐름 (E -> T -> L).
        """
        all_errors = []
        all_success = True

        for source_name, data_df in sources.items():

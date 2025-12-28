# Apache Airflow - Workflow Orchestration Platform

## Tổng quan về Apache Airflow

Apache Airflow là nền tảng mã nguồn mở để tạo, lập lịch và giám sát workflows. Airflow sử dụng Python để định nghĩa workflows dưới dạng Directed Acyclic Graphs (DAGs), cung cấp khả năng scheduling, monitoring và dependency management mạnh mẽ.

## Cài đặt và Cấu hình

### Local Installation
```bash
# Create virtual environment
python -m venv airflow-env
source airflow-env/bin/activate

# Set Airflow home
export AIRFLOW_HOME=~/airflow

# Install Airflow
AIRFLOW_VERSION=2.7.2
PYTHON_VERSION="$(python --version | cut -d " " -f 2 | cut -d "." -f 1-2)"
CONSTRAINT_URL="https://raw.githubusercontent.com/apache/airflow/constraints-${AIRFLOW_VERSION}/constraints-${PYTHON_VERSION}.txt"
pip install "apache-airflow==${AIRFLOW_VERSION}" --constraint "${CONSTRAINT_URL}"

# Initialize database
airflow db init

# Create admin user
airflow users create \
    --username admin \
    --firstname Admin \
    --lastname User \
    --role Admin \
    --email admin@example.com \
    --password admin

# Start webserver and scheduler
airflow webserver --port 8080 &
airflow scheduler &
```

### Docker Compose Setup
```yaml
# docker-compose.yml
version: '3.8'

x-airflow-common:
  &airflow-common
  image: apache/airflow:2.7.2
  environment: &airflow-common-env
    AIRFLOW__CORE__EXECUTOR: CeleryExecutor
    AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
    AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://airflow:airflow@postgres/airflow
    AIRFLOW__CELERY__BROKER_URL: redis://:@redis:6379/0
    AIRFLOW__CORE__FERNET_KEY: ''
    AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION: 'true'
    AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
    AIRFLOW__API__AUTH_BACKENDS: 'airflow.api.auth.backend.basic_auth,airflow.api.auth.backend.session'
    AIRFLOW__SCHEDULER__ENABLE_HEALTH_CHECK: 'true'
    _PIP_ADDITIONAL_REQUIREMENTS: ${_PIP_ADDITIONAL_REQUIREMENTS:-}
  volumes:
    - ${AIRFLOW_PROJ_DIR:-.}/dags:/opt/airflow/dags
    - ${AIRFLOW_PROJ_DIR:-.}/logs:/opt/airflow/logs
    - ${AIRFLOW_PROJ_DIR:-.}/plugins:/opt/airflow/plugins
    - ${AIRFLOW_PROJ_DIR:-.}/config:/opt/airflow/config
  user: "${AIRFLOW_UID:-50000}:0"
  depends_on: &airflow-common-depends-on
    redis:
      condition: service_healthy
    postgres:
      condition: service_healthy

services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow
    volumes:
      - postgres-db-volume:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "airflow"]
      interval: 10s
      retries: 5
      start_period: 5s
    restart: always

  redis:
    image: redis:latest
    expose:
      - 6379
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 30s
      retries: 50
      start_period: 30s
    restart: always

  airflow-webserver:
    <<: *airflow-common
    command: webserver
    ports:
      - "8080:8080"
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

  airflow-scheduler:
    <<: *airflow-common
    command: scheduler
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:8974/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

  airflow-worker:
    <<: *airflow-common
    command: celery worker
    healthcheck:
      test:
        - "CMD-SHELL"
        - 'celery --app airflow.executors.celery_executor.app inspect ping -d "celery@$${HOSTNAME}"'
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    environment:
      <<: *airflow-common-env
      DUMB_INIT_SETSID: "0"
    restart: always
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

  airflow-triggerer:
    <<: *airflow-common
    command: triggerer
    healthcheck:
      test: ["CMD-SHELL", 'airflow jobs check --job-type TriggererJob --hostname "$${HOSTNAME}"']
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

  airflow-init:
    <<: *airflow-common
    entrypoint: /bin/bash
    command:
      - -c
      - |
        function ver() {
          printf "%04d%04d%04d%04d" $${1//./ }
        }
        airflow_version=$$(AIRFLOW__LOGGING__LOGGING_LEVEL=INFO && airflow version)
        airflow_version_comparable=$$(ver $${airflow_version})
        min_airflow_version=2.2.0
        min_airflow_version_comparable=$$(ver $${min_airflow_version})
        if (( airflow_version_comparable < min_airflow_version_comparable )); then
          echo
          echo -e "\033[1;31mERROR!!!: Too old Airflow version $${airflow_version}!\e[0m"
          echo "The minimum Airflow version supported: $${min_airflow_version}. Only use this or higher!"
          echo
          exit 1
        fi
        if [[ -z "${AIRFLOW_UID}" ]]; then
          echo
          echo -e "\033[1;33mWARNING!!!: AIRFLOW_UID not set!\e[0m"
          echo "If you are on Linux, you SHOULD follow the instructions below to set "
          echo "AIRFLOW_UID environment variable, otherwise files will be owned by root."
          echo "For other operating systems you can get rid of the warning with manually created .env file:"
          echo "    See: https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html#setting-the-right-airflow-user"
          echo
        fi
        one_meg=1048576
        mem_available=$$(($$(getconf _PHYS_PAGES) * $$(getconf PAGE_SIZE) / one_meg))
        cpus_available=$$(grep -cE 'cpu[0-9]+' /proc/stat)
        disk_available=$$(df / | tail -1 | awk '{print $$4}')
        warning_resources="false"
        if (( mem_available < 4000 )) ; then
          echo
          echo -e "\033[1;33mWARNING!!!: Not enough memory available for Docker.\e[0m"
          echo "At least 4GB of memory required. You have $$(numfmt --to iec $$((mem_available * one_meg)))"
          echo
          warning_resources="true"
        fi
        if (( cpus_available < 2 )); then
          echo
          echo -e "\033[1;33mWARNING!!!: Not enough CPUS available for Docker.\e[0m"
          echo "At least 2 CPUs recommended. You have $${cpus_available}"
          echo
          warning_resources="true"
        fi
        if (( disk_available < one_meg )); then
          echo
          echo -e "\033[1;33mWARNING!!!: Not enough Disk space available for Docker.\e[0m"
          echo "At least 1 GiB recommended. You have $$(numfmt --to iec $$((disk_available * 1024 )))"
          echo
          warning_resources="true"
        fi
        if [[ $${warning_resources} == "true" ]]; then
          echo
          echo -e "\033[1;33mWARNING!!!: You have not enough resources to run Airflow (see above)!\e[0m"
          echo "Please follow the instructions to increase amount of resources available:"
          echo "   https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html#before-you-begin"
          echo
        fi
        mkdir -p /sources/logs /sources/dags /sources/plugins
        chown -R "${AIRFLOW_UID}:0" /sources/{logs,dags,plugins}
        exec /entrypoint airflow version
    environment:
      <<: *airflow-common-env
      _AIRFLOW_DB_UPGRADE: 'true'
      _AIRFLOW_WWW_USER_CREATE: 'true'
      _AIRFLOW_WWW_USER_USERNAME: ${_AIRFLOW_WWW_USER_USERNAME:-airflow}
      _AIRFLOW_WWW_USER_PASSWORD: ${_AIRFLOW_WWW_USER_PASSWORD:-airflow}
    user: "0:0"
    volumes:
      - ${AIRFLOW_PROJ_DIR:-.}:/sources

  flower:
    <<: *airflow-common
    command: celery flower
    profiles:
      - flower
    ports:
      - "5555:5555"
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:5555/"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

volumes:
  postgres-db-volume:
```

## DAG Development

### Basic DAG Structure
```python
# dags/basic_dag.py
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator
from airflow.operators.email import EmailOperator
from airflow.sensors.filesystem import FileSensor

# Default arguments
default_args = {
    'owner': 'data-team',
    'depends_on_past': False,
    'start_date': datetime(2023, 1, 1),
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
    'email': ['admin@company.com']
}

# Define DAG
dag = DAG(
    'data_processing_pipeline',
    default_args=default_args,
    description='Daily data processing pipeline',
    schedule_interval='0 2 * * *',  # Daily at 2 AM
    catchup=False,
    max_active_runs=1,
    tags=['data', 'etl', 'daily']
)

def extract_data(**context):
    """Extract data from source systems"""
    import pandas as pd
    import logging
    
    # Get execution date
    execution_date = context['execution_date']
    logging.info(f"Extracting data for {execution_date}")
    
    # Simulate data extraction
    data = {
        'id': range(1, 1001),
        'value': [i * 2 for i in range(1, 1001)],
        'date': [execution_date] * 1000
    }
    
    df = pd.DataFrame(data)
    
    # Save to temporary location
    output_path = f"/tmp/extracted_data_{execution_date.strftime('%Y%m%d')}.csv"
    df.to_csv(output_path, index=False)
    
    # Return path for next task
    return output_path

def transform_data(**context):
    """Transform extracted data"""
    import pandas as pd
    import logging
    
    # Get input path from previous task
    ti = context['task_instance']
    input_path = ti.xcom_pull(task_ids='extract_data')
    
    logging.info(f"Transforming data from {input_path}")
    
    # Load and transform data
    df = pd.read_csv(input_path)
    df['transformed_value'] = df['value'] * 1.5
    df['category'] = df['value'].apply(lambda x: 'high' if x > 1000 else 'low')
    
    # Save transformed data
    execution_date = context['execution_date']
    output_path = f"/tmp/transformed_data_{execution_date.strftime('%Y%m%d')}.csv"
    df.to_csv(output_path, index=False)
    
    return output_path

def load_data(**context):
    """Load data to destination"""
    import pandas as pd
    import logging
    
    # Get input path from previous task
    ti = context['task_instance']
    input_path = ti.xcom_pull(task_ids='transform_data')
    
    logging.info(f"Loading data from {input_path}")
    
    # Simulate loading to database
    df = pd.read_csv(input_path)
    logging.info(f"Loaded {len(df)} records to database")
    
    return len(df)

# File sensor to wait for input file
wait_for_file = FileSensor(
    task_id='wait_for_input_file',
    filepath='/data/input/{{ ds }}/data.txt',
    fs_conn_id='fs_default',
    poke_interval=60,
    timeout=300,
    dag=dag
)

# Extract task
extract_task = PythonOperator(
    task_id='extract_data',
    python_callable=extract_data,
    dag=dag
)

# Transform task
transform_task = PythonOperator(
    task_id='transform_data',
    python_callable=transform_data,
    dag=dag
)

# Load task
load_task = PythonOperator(
    task_id='load_data',
    python_callable=load_data,
    dag=dag
)

# Data quality check
quality_check = BashOperator(
    task_id='data_quality_check',
    bash_command='''
    echo "Running data quality checks..."
    # Add your data quality validation logic here
    python /opt/airflow/scripts/quality_check.py --date {{ ds }}
    ''',
    dag=dag
)

# Success notification
success_email = EmailOperator(
    task_id='send_success_email',
    to=['team@company.com'],
    subject='Data Pipeline Success - {{ ds }}',
    html_content='''
    <h3>Data Pipeline Completed Successfully</h3>
    <p>Date: {{ ds }}</p>
    <p>Records processed: {{ ti.xcom_pull(task_ids='load_data') }}</p>
    <p>Pipeline completed at: {{ ts }}</p>
    ''',
    dag=dag
)

# Define task dependencies
wait_for_file >> extract_task >> transform_task >> load_task >> quality_check >> success_email
```

### Advanced DAG with Dynamic Tasks
```python
# dags/dynamic_dag.py
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator
from airflow.models import Variable
import json

default_args = {
    'owner': 'data-team',
    'depends_on_past': False,
    'start_date': datetime(2023, 1, 1),
    'retries': 1,
    'retry_delay': timedelta(minutes=5)
}

dag = DAG(
    'dynamic_processing_dag',
    default_args=default_args,
    description='Dynamic task generation based on configuration',
    schedule_interval='0 3 * * *',
    catchup=False,
    tags=['dynamic', 'etl']
)

# Get configuration from Airflow Variables
config = json.loads(Variable.get("processing_config", default_var='''
{
    "sources": [
        {"name": "source1", "type": "database", "table": "users"},
        {"name": "source2", "type": "api", "endpoint": "/api/orders"},
        {"name": "source3", "type": "file", "path": "/data/products.csv"}
    ],
    "destinations": ["warehouse", "analytics", "reporting"]
}
'''))

def process_source(source_name, source_config, **context):
    """Process individual source"""
    import logging
    
    logging.info(f"Processing source: {source_name}")
    logging.info(f"Source config: {source_config}")
    
    if source_config['type'] == 'database':
        # Database processing logic
        logging.info(f"Extracting from table: {source_config['table']}")
    elif source_config['type'] == 'api':
        # API processing logic
        logging.info(f"Calling endpoint: {source_config['endpoint']}")
    elif source_config['type'] == 'file':
        # File processing logic
        logging.info(f"Reading file: {source_config['path']}")
    
    # Return processed data info
    return {
        'source': source_name,
        'records': 1000,  # Simulated record count
        'status': 'success'
    }

def aggregate_results(**context):
    """Aggregate results from all source processing tasks"""
    import logging
    
    ti = context['task_instance']
    total_records = 0
    
    # Get results from all source tasks
    for source in config['sources']:
        task_id = f"process_{source['name']}"
        result = ti.xcom_pull(task_ids=task_id)
        if result:
            total_records += result.get('records', 0)
            logging.info(f"Source {result['source']}: {result['records']} records")
    
    logging.info(f"Total records processed: {total_records}")
    return total_records

# Create dynamic tasks for each source
source_tasks = []
for source in config['sources']:
    task = PythonOperator(
        task_id=f"process_{source['name']}",
        python_callable=process_source,
        op_args=[source['name'], source],
        dag=dag
    )
    source_tasks.append(task)

# Aggregation task
aggregate_task = PythonOperator(
    task_id='aggregate_results',
    python_callable=aggregate_results,
    dag=dag
)

# Create dynamic tasks for each destination
destination_tasks = []
for dest in config['destinations']:
    task = BashOperator(
        task_id=f"load_to_{dest}",
        bash_command=f'''
        echo "Loading data to {dest}..."
        # Add destination-specific loading logic
        python /opt/airflow/scripts/load_to_{dest}.py --records {{{{ ti.xcom_pull(task_ids='aggregate_results') }}}}
        ''',
        dag=dag
    )
    destination_tasks.append(task)

# Set dependencies
for source_task in source_tasks:
    source_task >> aggregate_task

for dest_task in destination_tasks:
    aggregate_task >> dest_task
```

## Custom Operators

### Database Operator
```python
# plugins/operators/database_operator.py
from airflow.models import BaseOperator
from airflow.hooks.postgres_hook import PostgresHook
from airflow.utils.decorators import apply_defaults
import pandas as pd

class DatabaseETLOperator(BaseOperator):
    """
    Custom operator for database ETL operations
    """
    
    template_fields = ['sql', 'destination_table']
    
    @apply_defaults
    def __init__(
        self,
        sql,
        postgres_conn_id='postgres_default',
        destination_table=None,
        destination_conn_id=None,
        chunk_size=10000,
        *args,
        **kwargs
    ):
        super().__init__(*args, **kwargs)
        self.sql = sql
        self.postgres_conn_id = postgres_conn_id
        self.destination_table = destination_table
        self.destination_conn_id = destination_conn_id or postgres_conn_id
        self.chunk_size = chunk_size
    
    def execute(self, context):
        # Source hook
        source_hook = PostgresHook(postgres_conn_id=self.postgres_conn_id)
        
        # Destination hook
        dest_hook = PostgresHook(postgres_conn_id=self.destination_conn_id)
        
        self.log.info(f"Executing SQL: {self.sql}")
        
        # Execute query and get results
        df = source_hook.get_pandas_df(self.sql)
        
        self.log.info(f"Retrieved {len(df)} records")
        
        if self.destination_table:
            # Insert data in chunks
            total_inserted = 0
            for chunk_start in range(0, len(df), self.chunk_size):
                chunk_end = min(chunk_start + self.chunk_size, len(df))
                chunk_df = df.iloc[chunk_start:chunk_end]
                
                # Convert DataFrame to list of tuples
                records = [tuple(row) for row in chunk_df.values]
                
                # Insert chunk
                dest_hook.insert_rows(
                    table=self.destination_table,
                    rows=records,
                    target_fields=list(chunk_df.columns)
                )
                
                total_inserted += len(records)
                self.log.info(f"Inserted chunk: {len(records)} records")
            
            self.log.info(f"Total inserted: {total_inserted} records")
            return total_inserted
        
        return len(df)

# Usage in DAG
from plugins.operators.database_operator import DatabaseETLOperator

etl_task = DatabaseETLOperator(
    task_id='extract_transform_load',
    sql='''
    SELECT 
        user_id,
        email,
        created_at,
        CASE 
            WHEN created_at >= '{{ ds }}' THEN 'new'
            ELSE 'existing'
        END as user_type
    FROM users 
    WHERE created_at >= '{{ ds }}'
    ''',
    destination_table='processed_users',
    postgres_conn_id='source_db',
    destination_conn_id='warehouse_db',
    dag=dag
)
```

### API Operator
```python
# plugins/operators/api_operator.py
from airflow.models import BaseOperator
from airflow.hooks.http_hook import HttpHook
from airflow.utils.decorators import apply_defaults
import json
import pandas as pd

class APIExtractOperator(BaseOperator):
    """
    Custom operator for API data extraction
    """
    
    template_fields = ['endpoint', 'data', 'headers']
    
    @apply_defaults
    def __init__(
        self,
        endpoint,
        method='GET',
        data=None,
        headers=None,
        http_conn_id='http_default',
        pagination=False,
        page_size=100,
        max_pages=None,
        *args,
        **kwargs
    ):
        super().__init__(*args, **kwargs)
        self.endpoint = endpoint
        self.method = method
        self.data = data
        self.headers = headers or {}
        self.http_conn_id = http_conn_id
        self.pagination = pagination
        self.page_size = page_size
        self.max_pages = max_pages
    
    def execute(self, context):
        http_hook = HttpHook(method=self.method, http_conn_id=self.http_conn_id)
        
        all_data = []
        page = 1
        
        while True:
            # Prepare request parameters
            if self.pagination:
                params = {'page': page, 'limit': self.page_size}
                if self.data:
                    params.update(self.data)
            else:
                params = self.data
            
            self.log.info(f"Making API request to {self.endpoint}, page {page}")
            
            # Make API request
            response = http_hook.run(
                endpoint=self.endpoint,
                data=json.dumps(params) if params else None,
                headers=self.headers
            )
            
            if response.status_code != 200:
                raise Exception(f"API request failed: {response.status_code} - {response.text}")
            
            response_data = response.json()
            
            # Extract data based on response structure
            if isinstance(response_data, list):
                page_data = response_data
            elif 'data' in response_data:
                page_data = response_data['data']
            elif 'results' in response_data:
                page_data = response_data['results']
            else:
                page_data = [response_data]
            
            all_data.extend(page_data)
            
            self.log.info(f"Retrieved {len(page_data)} records from page {page}")
            
            # Check if we should continue pagination
            if not self.pagination:
                break
            
            if len(page_data) < self.page_size:
                self.log.info("Reached end of data")
                break
            
            if self.max_pages and page >= self.max_pages:
                self.log.info(f"Reached maximum pages limit: {self.max_pages}")
                break
            
            page += 1
        
        self.log.info(f"Total records retrieved: {len(all_data)}")
        
        # Convert to DataFrame for easier processing
        df = pd.DataFrame(all_data)
        
        # Save to temporary location
        execution_date = context['execution_date']
        output_path = f"/tmp/api_data_{execution_date.strftime('%Y%m%d_%H%M%S')}.json"
        
        with open(output_path, 'w') as f:
            json.dump(all_data, f)
        
        return {
            'records_count': len(all_data),
            'output_path': output_path,
            'columns': list(df.columns) if not df.empty else []
        }

# Usage in DAG
api_extract = APIExtractOperator(
    task_id='extract_from_api',
    endpoint='/api/v1/users',
    method='GET',
    headers={'Authorization': 'Bearer {{ var.value.api_token }}'},
    pagination=True,
    page_size=1000,
    max_pages=10,
    http_conn_id='external_api',
    dag=dag
)
```

## Sensors và Hooks

### Custom File Sensor
```python
# plugins/sensors/s3_file_sensor.py
from airflow.sensors.base import BaseSensorOperator
from airflow.hooks.S3_hook import S3Hook
from airflow.utils.decorators import apply_defaults
from datetime import datetime

class S3FileSensor(BaseSensorOperator):
    """
    Sensor to check for file existence in S3
    """
    
    template_fields = ['bucket_name', 'key']
    
    @apply_defaults
    def __init__(
        self,
        bucket_name,
        key,
        aws_conn_id='aws_default',
        verify=None,
        *args,
        **kwargs
    ):
        super().__init__(*args, **kwargs)
        self.bucket_name = bucket_name
        self.key = key
        self.aws_conn_id = aws_conn_id
        self.verify = verify
    
    def poke(self, context):
        """
        Check if file exists in S3
        """
        s3_hook = S3Hook(aws_conn_id=self.aws_conn_id, verify=self.verify)
        
        self.log.info(f"Checking for file: s3://{self.bucket_name}/{self.key}")
        
        exists = s3_hook.check_for_key(key=self.key, bucket_name=self.bucket_name)
        
        if exists:
            self.log.info(f"File found: s3://{self.bucket_name}/{self.key}")
            return True
        else:
            self.log.info(f"File not found: s3://{self.bucket_name}/{self.key}")
            return False

# Custom Database Sensor
class DatabaseRecordSensor(BaseSensorOperator):
    """
    Sensor to check for new records in database
    """
    
    template_fields = ['sql']
    
    @apply_defaults
    def __init__(
        self,
        sql,
        postgres_conn_id='postgres_default',
        min_records=1,
        *args,
        **kwargs
    ):
        super().__init__(*args, **kwargs)
        self.sql = sql
        self.postgres_conn_id = postgres_conn_id
        self.min_records = min_records
    
    def poke(self, context):
        from airflow.hooks.postgres_hook import PostgresHook
        
        postgres_hook = PostgresHook(postgres_conn_id=self.postgres_conn_id)
        
        self.log.info(f"Executing SQL: {self.sql}")
        
        records = postgres_hook.get_records(self.sql)
        record_count = records[0][0] if records else 0
        
        self.log.info(f"Found {record_count} records")
        
        if record_count >= self.min_records:
            self.log.info(f"Condition met: {record_count} >= {self.min_records}")
            return True
        else:
            self.log.info(f"Condition not met: {record_count} < {self.min_records}")
            return False

# Usage in DAG
wait_for_s3_file = S3FileSensor(
    task_id='wait_for_input_file',
    bucket_name='my-data-bucket',
    key='input/{{ ds }}/data.csv',
    aws_conn_id='aws_default',
    poke_interval=60,
    timeout=300,
    dag=dag
)

wait_for_new_records = DatabaseRecordSensor(
    task_id='wait_for_new_records',
    sql='''
    SELECT COUNT(*) 
    FROM transactions 
    WHERE created_at >= '{{ ds }}' 
    AND processed = false
    ''',
    postgres_conn_id='source_db',
    min_records=100,
    poke_interval=120,
    timeout=600,
    dag=dag
)
```

## TaskGroups và SubDAGs

### TaskGroup Implementation
```python
# dags/taskgroup_dag.py
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator
from airflow.utils.task_group import TaskGroup

default_args = {
    'owner': 'data-team',
    'depends_on_past': False,
    'start_date': datetime(2023, 1, 1),
    'retries': 1,
    'retry_delay': timedelta(minutes=5)
}

dag = DAG(
    'taskgroup_example',
    default_args=default_args,
    description='Example DAG with TaskGroups',
    schedule_interval='0 4 * * *',
    catchup=False,
    tags=['taskgroup', 'etl']
)

def extract_source(source_name, **context):
    """Extract data from a specific source"""
    import logging
    logging.info(f"Extracting data from {source_name}")
    return f"data_from_{source_name}"

def transform_data(transformation_type, **context):
    """Transform data with specific transformation"""
    import logging
    ti = context['task_instance']
    
    # Get data from extract tasks
    data_sources = []
    for task_id in ['extract_database', 'extract_api', 'extract_files']:
        data = ti.xcom_pull(task_ids=f'extraction_group.{task_id}')
        if data:
            data_sources.append(data)
    
    logging.info(f"Applying {transformation_type} to {len(data_sources)} sources")
    return f"transformed_data_{transformation_type}"

def load_to_destination(destination, **context):
    """Load data to specific destination"""
    import logging
    ti = context['task_instance']
    
    # Get transformed data
    transformed_data = ti.xcom_pull(task_ids='transformation_group.apply_business_rules')
    
    logging.info(f"Loading {transformed_data} to {destination}")
    return f"loaded_to_{destination}"

# Extraction TaskGroup
with TaskGroup("extraction_group", dag=dag) as extraction_group:
    extract_db = PythonOperator(
        task_id='extract_database',
        python_callable=extract_source,
        op_args=['database']
    )
    
    extract_api = PythonOperator(
        task_id='extract_api',
        python_callable=extract_source,
        op_args=['api']
    )
    
    extract_files = PythonOperator(
        task_id='extract_files',
        python_callable=extract_source,
        op_args=['files']
    )

# Transformation TaskGroup
with TaskGroup("transformation_group", dag=dag) as transformation_group:
    clean_data = PythonOperator(
        task_id='clean_data',
        python_callable=transform_data,
        op_args=['cleaning']
    )
    
    validate_data = PythonOperator(
        task_id='validate_data',
        python_callable=transform_data,
        op_args=['validation']
    )
    
    apply_business_rules = PythonOperator(
        task_id='apply_business_rules',
        python_callable=transform_data,
        op_args=['business_rules']
    )
    
    # Dependencies within transformation group
    clean_data >> validate_data >> apply_business_rules

# Loading TaskGroup
with TaskGroup("loading_group", dag=dag) as loading_group:
    load_warehouse = PythonOperator(
        task_id='load_warehouse',
        python_callable=load_to_destination,
        op_args=['warehouse']
    )
    
    load_analytics = PythonOperator(
        task_id='load_analytics',
        python_callable=load_to_destination,
        op_args=['analytics']
    )
    
    load_reporting = PythonOperator(
        task_id='load_reporting',
        python_callable=load_to_destination,
        op_args=['reporting']
    )

# Data Quality TaskGroup
with TaskGroup("quality_group", dag=dag) as quality_group:
    row_count_check = BashOperator(
        task_id='row_count_check',
        bash_command='echo "Checking row counts..."'
    )
    
    data_freshness_check = BashOperator(
        task_id='data_freshness_check',
        bash_command='echo "Checking data freshness..."'
    )
    
    schema_validation = BashOperator(
        task_id='schema_validation',
        bash_command='echo "Validating schema..."'
    )

# Set TaskGroup dependencies
extraction_group >> transformation_group >> loading_group >> quality_group
```

## Monitoring và Alerting

### Custom Alerting
```python
# plugins/callbacks/alert_callbacks.py
from airflow.models import Variable
import requests
import json

def slack_alert_callback(context):
    """
    Send Slack notification on task failure
    """
    slack_webhook_url = Variable.get("slack_webhook_url")
    
    task_instance = context.get('task_instance')
    dag_id = context.get('dag').dag_id
    task_id = task_instance.task_id
    execution_date = context.get('execution_date')
    log_url = task_instance.log_url
    
    message = {
        "text": f"❌ Airflow Task Failed",
        "attachments": [
            {
                "color": "danger",
                "fields": [
                    {"title": "DAG", "value": dag_id, "short": True},
                    {"title": "Task", "value": task_id, "short": True},
                    {"title": "Execution Date", "value": str(execution_date), "short": True},
                    {"title": "Log URL", "value": log_url, "short": False}
                ]
            }
        ]
    }
    
    response = requests.post(
        slack_webhook_url,
        data=json.dumps(message),
        headers={'Content-Type': 'application/json'}
    )
    
    if response.status_code != 200:
        print(f"Failed to send Slack notification: {response.status_code}")

def teams_alert_callback(context):
    """
    Send Microsoft Teams notification on task failure
    """
    teams_webhook_url = Variable.get("teams_webhook_url")
    
    task_instance = context.get('task_instance')
    dag_id = context.get('dag').dag_id
    task_id = task_instance.task_id
    execution_date = context.get('execution_date')
    
    message = {
        "@type": "MessageCard",
        "@context": "http://schema.org/extensions",
        "themeColor": "FF0000",
        "summary": "Airflow Task Failed",
        "sections": [{
            "activityTitle": "Airflow Task Failed",
            "activitySubtitle": f"DAG: {dag_id}",
            "facts": [
                {"name": "Task", "value": task_id},
                {"name": "Execution Date", "value": str(execution_date)},
                {"name": "Status", "value": "Failed"}
            ],
            "markdown": True
        }]
    }
    
    response = requests.post(
        teams_webhook_url,
        data=json.dumps(message),
        headers={'Content-Type': 'application/json'}
    )

# Usage in DAG
from plugins.callbacks.alert_callbacks import slack_alert_callback

dag = DAG(
    'monitored_dag',
    default_args={
        'owner': 'data-team',
        'start_date': datetime(2023, 1, 1),
        'on_failure_callback': slack_alert_callback
    },
    schedule_interval='0 5 * * *'
)
```

### Health Check DAG
```python
# dags/health_check_dag.py
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator
from airflow.hooks.postgres_hook import PostgresHook
from airflow.hooks.http_hook import HttpHook

default_args = {
    'owner': 'ops-team',
    'depends_on_past': False,
    'start_date': datetime(2023, 1, 1),
    'retries': 1,
    'retry_delay': timedelta(minutes=2)
}

dag = DAG(
    'system_health_check',
    default_args=default_args,
    description='System health monitoring',
    schedule_interval='*/15 * * * *',  # Every 15 minutes
    catchup=False,
    tags=['monitoring', 'health']
)

def check_database_connection(**context):
    """Check database connectivity and performance"""
    import time
    
    postgres_hook = PostgresHook(postgres_conn_id='postgres_default')
    
    # Test connection
    start_time = time.time()
    try:
        records = postgres_hook.get_records("SELECT 1")
        connection_time = time.time() - start_time
        
        if connection_time > 5:  # 5 seconds threshold
            raise Exception(f"Database connection slow: {connection_time:.2f}s")
        
        return {
            'status': 'healthy',
            'connection_time': connection_time,
            'timestamp': datetime.now().isoformat()
        }
    except Exception as e:
        raise Exception(f"Database connection failed: {str(e)}")

def check_api_endpoints(**context):
    """Check critical API endpoints"""
    http_hook = HttpHook(method='GET', http_conn_id='api_default')
    
    endpoints = ['/health', '/api/v1/status', '/metrics']
    results = {}
    
    for endpoint in endpoints:
        try:
            response = http_hook.run(endpoint=endpoint)
            if response.status_code == 200:
                results[endpoint] = 'healthy'
            else:
                results[endpoint] = f'unhealthy: {response.status_code}'
        except Exception as e:
            results[endpoint] = f'error: {str(e)}'
    
    # Check if any endpoint is unhealthy
    unhealthy = [ep for ep, status in results.items() if 'unhealthy' in status or 'error' in status]
    
    if unhealthy:
        raise Exception(f"Unhealthy endpoints: {unhealthy}")
    
    return results

def check_disk_space(**context):
    """Check available disk space"""
    import shutil
    
    paths = ['/tmp', '/opt/airflow/logs', '/opt/airflow/dags']
    results = {}
    
    for path in paths:
        try:
            total, used, free = shutil.disk_usage(path)
            free_percent = (free / total) * 100
            
            results[path] = {
                'free_percent': free_percent,
                'free_gb': free / (1024**3)
            }
            
            if free_percent < 10:  # Less than 10% free
                raise Exception(f"Low disk space on {path}: {free_percent:.1f}% free")
                
        except Exception as e:
            results[path] = f"error: {str(e)}"
    
    return results

# Health check tasks
db_check = PythonOperator(
    task_id='check_database',
    python_callable=check_database_connection,
    dag=dag
)

api_check = PythonOperator(
    task_id='check_api_endpoints',
    python_callable=check_api_endpoints,
    dag=dag
)

disk_check = PythonOperator(
    task_id='check_disk_space',
    python_callable=check_disk_space,
    dag=dag
)

memory_check = BashOperator(
    task_id='check_memory_usage',
    bash_command='''
    MEMORY_USAGE=$(free | grep Mem | awk '{printf "%.1f", $3/$2 * 100.0}')
    echo "Memory usage: ${MEMORY_USAGE}%"
    
    if (( $(echo "$MEMORY_USAGE > 90" | bc -l) )); then
        echo "High memory usage: ${MEMORY_USAGE}%"
        exit 1
    fi
    ''',
    dag=dag
)

# Run all checks in parallel
[db_check, api_check, disk_check, memory_check]
```

Apache Airflow cung cấp một nền tảng workflow orchestration mạnh mẽ và linh hoạt với khả năng scheduling, monitoring và dependency management, phù hợp cho các quy trình ETL, data pipeline và automation workflows phức tạp.
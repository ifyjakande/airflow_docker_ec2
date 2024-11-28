# Complete Guide: Installing Apache Airflow with Docker on Ubuntu

This guide provides detailed, step-by-step instructions for setting up Apache Airflow in a production environment using Docker and PostgreSQL.

## Prerequisites

- AWS EC2 Ubuntu instance (minimum recommendation: t3.large with 30GB EBS)
- SSH access to your EC2 instance
- The following ports open in your security group:
  - 22 (SSH)
  - 8080 (Airflow webserver)
  - 5432 (PostgreSQL)
  - 5555 (Flower - if using Celery executor)

## Step 1: Initial Server Setup

First, connect to your EC2 instance:
```bash
ssh -i your-key.pem ubuntu@your-instance-ip
```

Update the system and install required packages:
```bash
sudo apt update
sudo apt install -y \
    python3-pip \
    python3-dev \
    python3-venv \
    build-essential \
    libpq-dev \
    docker.io \
    docker-compose
```

## Step 2: Configure Docker

Set up Docker permissions and start the service:
```bash
# Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# Add your user to the Docker group
sudo usermod -aG docker $USER

# Apply group changes
newgrp docker
```

## Step 3: Create Airflow Directory Structure

Create the necessary directories:
```bash
mkdir -p ~/airflow/{dags,logs,plugins,config}
cd ~/airflow
```

## Step 4: Set Up Python Virtual Environment

Create and configure a virtual environment:
```bash
# Create virtual environment
python3 -m venv ~/airflow_env

# Create activation script
cat << EOF > ~/activate_airflow
#!/bin/bash
source ~/airflow_env/bin/activate
cd ~/airflow
EOF

chmod +x ~/activate_airflow

# Activate and install requirements
source ~/airflow_env/bin/activate
pip install --upgrade pip
pip install apache-airflow-client psycopg2-binary pandas
```

## Step 5: Configure Environment Variables

Create the .env file with secure defaults:
```bash
cat << EOF > ~/airflow/.env
AIRFLOW_UID=$(id -u)
AIRFLOW_GID=0
POSTGRES_USER=airflow
POSTGRES_PASSWORD=$(openssl rand -hex 16)
POSTGRES_DB=airflow
AIRFLOW__CORE__SQL_ALCHEMY_CONN=postgresql+psycopg2://airflow:${POSTGRES_PASSWORD}@postgres/airflow
AIRFLOW__CORE__EXECUTOR=LocalExecutor
AIRFLOW__WEBSERVER__SECRET_KEY=$(openssl rand -hex 32)
AIRFLOW__CORE__FERNET_KEY=$(python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())")
EOF
```

## Step 6: Set Up Docker Compose

Download and modify the official Docker Compose file:
```bash
curl -LfO 'https://airflow.apache.org/docs/apache-airflow/stable/docker-compose.yaml'

# Switch to LocalExecutor
sed -i 's/CeleryExecutor/LocalExecutor/g' docker-compose.yaml
```

## Step 7: Set Secure Permissions

Apply proper ownership and permissions:
```bash
# Set ownership
sudo chown -R $USER:docker ~/airflow

# Set secure permissions
sudo chmod -R 750 ~/airflow
```

## Step 8: Initialize and Start Airflow

Initialize the Airflow database and start services:
```bash
# Initialize database
docker-compose up airflow-init

# Start all services
docker-compose up -d
```

## Step 9: Create First DAG

Create a test DAG to verify the setup:
```python
# ~/airflow/dags/test_dag.py
from airflow import DAG
from airflow.operators.python_operator import PythonOperator
from datetime import datetime, timedelta

def test_function():
    print("Airflow is working!")

default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}

dag = DAG(
    'test_dag',
    default_args=default_args,
    description='Test DAG',
    schedule_interval=timedelta(days=1),
)

test_task = PythonOperator(
    task_id='test_task',
    python_callable=test_function,
    dag=dag,
)
```

## Step 10: Access and Security Configuration

### Set Up Admin User
```bash
# Create admin user
docker-compose exec airflow-webserver airflow users create \
    --username admin \
    --firstname Admin \
    --lastname User \
    --role Admin \
    --email admin@example.com \
    --password $(openssl rand -hex 16)
```

### Configure RBAC (Role-Based Access Control)
Add to your .env file:
```bash
AIRFLOW__WEBSERVER__RBAC=True
AIRFLOW__WEBSERVER__AUTHENTICATE=True
```

## Production Security Measures

1. **SSL/TLS Configuration**
   Add to your .env file:
   ```bash
   AIRFLOW__WEBSERVER__WEB_SERVER_SSL_CERT=/path/to/cert
   AIRFLOW__WEBSERVER__WEB_SERVER_SSL_KEY=/path/to/key
   ```

2. **Configure Logging**
   Add to your .env file:
   ```bash
   AIRFLOW__LOGGING__BASE_LOG_FOLDER=/opt/airflow/logs
   AIRFLOW__LOGGING__DAG_FILE_PROCESSOR_LOG_TARGET=/opt/airflow/logs/dag_processor_manager/dag_processor_manager.log
   ```

## Maintenance Tasks

### Backup Strategy
```bash
# Backup DAGs
tar -czf airflow_dags_backup.tar.gz ~/airflow/dags

# Backup PostgreSQL database
docker-compose exec postgres pg_dump -U airflow airflow > airflow_db_backup.sql
```

### Update Airflow
```bash
# Pull new images
docker-compose pull

# Restart services
docker-compose down
docker-compose up -d
```

## Troubleshooting

1. **Permission Issues**
   ```bash
   # Fix Docker socket permissions
   sudo chmod 666 /var/run/docker.sock
   ```

2. **Service Status**
   ```bash
   # Check running services
   docker-compose ps
   
   # View logs
   docker-compose logs airflow-webserver
   ```

3. **Reset Environment**
   ```bash
   # Complete reset
   docker-compose down -v
   docker-compose up -d
   ```

## Monitoring

1. **Container Health**
   ```bash
   # Monitor container resources
   docker stats
   ```

2. **Log Monitoring**
   ```bash
   # Real-time log monitoring
   docker-compose logs -f
   ```

## Best Practices

1. **Regular Updates**
   - Keep Docker images updated
   - Regularly update Python packages
   - Monitor security advisories

2. **Backup Strategy**
   - Daily database backups
   - Version control for DAGs
   - Regular configuration backups

3. **Security**
   - Rotate passwords regularly
   - Use AWS Secrets Manager for credentials
   - Implement network segmentation
   - Regular security audits

4. **Performance**
   - Monitor resource usage
   - Optimize DAG scheduling
   - Regular database maintenance

Remember to replace placeholder values (passwords, certificates) with your actual secure values in production.

## Additional Resources

- [Official Airflow Documentation](https://airflow.apache.org/docs/)
- [Docker Documentation](https://docs.docker.com/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)

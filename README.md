# Ansible Airflow

The following will configure an AlmaLinux 9 instance with Docker, Airflow 2.11.0 and Postgres 16.9.

Defaults: 

* The Airflow application data will be in /opt/airflow
* You can put Postgres database dumps into /opt/postgres-backups to have them available inside the container for restore

# Prep

Ensure that:

* You have a user with SSH authorized_keys on the destination instance
* The user can execute sudo without a password on the destination instance
* [Install Ansible](https://docs.ansible.com/ansible/latest/installation_guide/installation_distros.html) on your local machine
* Adjust the inventory.ini to match your environment (IP, username, SSH private key)

# Automated Provisioning

```bash
# check that Ansible is installed, configured and can reach the destination instance
ansible -i inventory.ini all -m ping
```

```bash
# run the Ansible playbook
ansible-playbook -v -i inventory.ini playbook.yml
 ```
# Manual Provisioning Steps

1. Ensure system packages are up to date

```bash
dnf update -y
```

2. Install required packages

```bash
dnf install -y yum-utils device-mapper-persistent-data lvm2
```

3. Add Docker repository

```bash
# download https://download.docker.com/linux/centos/docker-ce.repo to /etc/yum.repos.d/docker-ce.repo
```

4. Disable * repo

```bash
# set enabled=0 for all listed repositories (required on ARM64)
```

5. Install Docker

```bash
dnf update
dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin 
```

6. Enable and start Docker service

```bash
systemctl enable docker.service
systemctl start docker.service
```

7. Add current user to docker group (required for Ansible)

```bash
# add the ansible automation user to the docker group
```

8. Airflow group, directory and permissions

```bash
# add airflow group with GID 50000
# add airflow user with UID 50000, group airflow and no home directory
mkdir -p /opt/airflow/logs
chown -R airflow: /opt/airflow
chmod 0755 /opt/airflow
```

9. Custom Airflow Dockerfile `/opt/airflow/Dockerfile`

```bash
FROM apache/airflow:2.11.0

USER root

RUN ln -s /opt/airflow /home/airflow/airflow
RUN groupadd -g 996 docker \
  && usermod -aG docker airflow

USER airflow
```

10. Create Docker Compose configuration file `/opt/docker-compose.yml`

```yaml
x-airflow-common:
  &airflow-common
  build: ./airflow
  container_name: airflow-scheduler
  restart: always
  depends_on:
    - postgres
  environment:
    AIRFLOW__CELERY__BROKER_URL: "redis://redis:6379/0"
    AIRFLOW__CELERY__RESULT_BACKEND: "db+postgresql://airflow:airflow@postgres/airflow"
    AIRFLOW__CORE__LOAD_EXAMPLES: "False"
    AIRFLOW__CORE__EXECUTOR: "CeleryExecutor"
    AIRFLOW__CORE__DEFAULT_TIMEZONE: "Pacific/Auckland"
    AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: "postgresql+psycopg2://airflow:airflow@postgres/airflow"
    AIRFLOW__EMAIL_EMAIL_BACKEND: "airflow.utils.email.send_email_smtp"
    AIRFLOW__EMAIL__DEFAULT_EMAIL_ON_FAILURE: "True"
    AIRFLOW__EMAIL__DEFAULT_EMAIL_ON_RETRY: "True"
    AIRFLOW__SCHEDULER__CATCHUP_BY_DEFAULT: "False"
    AIRFLOW__WEBSERVER__DEFAULT_UI_TIMEZONE: "Pacific/Auckland"
    AIRFLOW__WEBSERVER__EXPOSE_CONFIG: "True"
    TZ: "Pacific/Auckland"
  volumes:
    - /opt/airflow:/opt/airflow
    - /var/run/docker.sock:/var/run/docker.sock

  services:
    postgres:
      image: postgres:16.9
      container_name: postgres
      restart: always
      ports:
        - "5432:5432"
      environment:
        POSTGRES_USER: airflow
        POSTGRES_PASSWORD: airflow
        POSTGRES_DB: airflow
        TZ: "Pacific/Auckland"
      healthcheck:
        test: ["CMD", "pg_isready", "-U", "airflow"]
        interval: 10s
        retries: 5
        start_period: 5s
      volumes:
        - postgres_data:/var/lib/postgresql/data
        - /opt/postgres-backups:/opt/postgres-backups

    redis:
      # Redis is limited to 7.2-bookworm due to licencing change
      # https://redis.io/blog/redis-adopts-dual-source-available-licensing/
      image: redis:7.2-bookworm
      container_name: redis
      expose:
        - 6379
      healthcheck:
        test: ["CMD", "redis-cli", "ping"]
        interval: 10s
        timeout: 30s
        retries: 50
        start_period: 30s
      restart: always

    airflow-scheduler:
      <<: *airflow-common
      container_name: airflow-scheduler
      command: scheduler
      healthcheck:
        test: ["CMD-SHELL", "airflow jobs check --job-type SchedulerJob --hostname $(hostname)"]
        interval: 30s
        timeout: 10s
        retries: 5
        start_period: 30s

    airflow-webserver:
      <<: *airflow-common
      container_name: airflow-webserver
      ports:
        - "8080:8080"
      command: webserver
      healthcheck:
        test: ["CMD", "curl", "--fail", "http://localhost:8080/airflow/health"]
        interval: 30s
        timeout: 10s
        retries: 5
        start_period: 30s

  volumes:
    postgres_data:
```

11. Launch containers

```bash
cd /opt
docker compose up -d
```

12. Open port 8080/tcp in firewalld

# Manual Tasks

### Perform initial Airflow init/bootstrap and DB migration

```bash
cd /opt
docker compose run --rm -it airflow db migrate
```

### Create an admin user for testing  

```bash
cd /opt
docker compose run --rm -it airflow users create --role Admin --username admin --email admin --firstname admin --lastname admin --password admin
```

### Backup existing Airflow data

```bash
cd ~airflow/airflow
tar --exclude "logs/*" -czvf ~/airflow-data.tgz .
```

### Restore an Airflow Postgres database dump

```bash
cd /opt
docker exec -it postgres sh -c "gunzip -cd /opt/postgres-backups/airflow.db.gz | psql -U airflow -W -h localhost -d airflow"
```

### Execute Python to convert dags to newer format
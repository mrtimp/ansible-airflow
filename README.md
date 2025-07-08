# Ansible Airflow

The following will configure an AlmaLinux 9 instance with Docker, Airflow 2.10.5 and Postgres 16.9.

# Prep

Ensure that:

* You have a user with SSH authorized_keys on the destination instance
* The user can execute sudo without a password on the destination instance
* [Install Ansible](https://docs.ansible.com/ansible/latest/installation_guide/installation_distros.html) on your local machine
* Adjust the inventory.ini to match your environment (IP, username, SSH private key)

# Automated Provisioning

```bash
# check that Ansible is installed, configured and can reach the destination host
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

9. Create Docker Compose configuration file `/opt/docker-compose.yml`

```yaml
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
    volumes:
      - postgres_data:/var/lib/postgresql/data
    
  airflow:
    image: apache/airflow:2.10.5
    container_name: airflow
    restart: always
    ports:
      - "8080:8080"
    depends_on:
      - postgres
    environment:
      TZ: "Pacific/Auckland"
    
      AIRFLOW__CORE__LOAD_EXAMPLES: "False"
      #AIRFLOW__CORE__SQL_ALCHEMY_CONN: "postgresql+psycopg2://airflow:airflow@postgres/airflow"
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: "postgresql+psycopg2://airflow:airflow@postgres/airflow"
    
      AIRFLOW__CORE__EXECUTOR: "CeleryExecutor"
      #AIRFLOW__UID: "50000"
      AIRFLOW__CORE__DEFAULT_TIMEZONE: "Pacific/Auckland"
      AIRFLOW__WEBSERVER__DEFAULT_UI_TIMEZONE: "Pacific/Auckland"
      #AIRFLOW__CELERY__BROKER_URL: "redis://${var.redis_address}:6379/0"
      AIRFLOW__WEBSERVER__EXPOSE_CONFIG: "True"
      #AIRFLOW__EMAIL__HTML_CONTENT_TEMPLATE: "/opt/airflow/email_templates/email-content-template.html"
      #AIRFLOW__EMAIL__HTML_SUBJECT_TEMPLATE: "/opt/airflow/email_templates/email-subject-template.html"
      AIRFLOW__SCHEDULER__CATCHUP_BY_DEFAULT: "False"
      AIRFLOW__EMAIL_EMAIL_BACKEND: "airflow.utils.email.send_email_smtp"
      AIRFLOW__EMAIL__DEFAULT_EMAIL_ON_FAILURE: "True"
      AIRFLOW__EMAIL__DEFAULT_EMAIL_ON_RETRY: "True"
      #AIRFLOW__SMTP__SMTP_HOST: "email-smtp.ap-southeast-2.amazonaws.com"
      #AIRFLOW__SMTP__SMTP_MAIL_FROM: "Airflow<noreply@example.com>"
      #AIRFLOW__SMTP__SMTP_PORT: "587"
      #AIRFLOW__SMTP__SMTP_STARTTLS: "True"
      #AIRFLOW__SMTP__SMTP_SSL: "False"
    volumes:
      - /opt/airflow:/opt/airflow
    command: webserver
    
 volumes:
   postgres_data:
```

10. Launch containers

```bash
cd /opt
docker compose up -d
```

11. Open port 8080/tcp in firewalld
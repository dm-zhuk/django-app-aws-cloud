# Інструкція з деплою та налаштувань

## Ця інструкція описує процес деплою Django polls додатку на AWS з використанням ECS Fargate, ALB, RDS, ECR, CloudWatch та інших компонентів, а також налаштування для забезпечення функціональності.

### Підготовка вихідного коду (Mac Terminal):

git clone https://github.com/dm-zhuk/django-app-aws-cloud.git
cd django-app-aws-cloud/django_app

> requirements.txt містить необхідні залежності:

Django==4.2.23
psycopg2-binary==2.9.10
gunicorn==23.0.0
whitenoise==6.8.2

Налаштуйте django_app/settings.py:

ALLOWED_HOSTS = ['internal-fp-072025-alb-1020846421.eu-central-1.elb.amazonaws.com', 'fp-072025-alb-public-949119113.eu-central-1.elb.amazonaws.com', 'localhost', '127.0.0.1', '*']
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'django_db',
        'USER': 'postgres',
        'PASSWORD': '127devopsql5432',
        'HOST': 'django-db.c9ag0ssuo6fe.eu-central-1.rds.amazonaws.com',
        'PORT': '5432',
    }
}
STATIC_ROOT = BASE_DIR / 'staticfiles'
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'

Зберіть статичні файли:
python3 manage.py collectstatic

### Створення Docker-образу та завантаження в ECR (Mac Terminal):
Створіть Dockerfile:dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
RUN python manage.py collectstatic --noinput
EXPOSE 8000
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "django_app.wsgi:application"]

Побудуйте та завантажте образ:
docker build --no-cache -t django-app .
docker tag django-app:latest 381492090902.dkr.ecr.eu-central-1.amazonaws.com/django-app:latest
aws ecr get-login-password --region eu-central-1 | docker login --username AWS --password-stdin 381492090902.dkr.ecr.eu-central-1.amazonaws.com
docker push 381492090902.dkr.ecr.eu-central-1.amazonaws.com/django-app:latest

### Налаштування ECS Fargate (AWS Console, AWS CloudShell):
Створіть кластер ECS (AWS Console):ECS > Clusters > Create Cluster.
Тип: Networking only (Fargate), Назва: fp-072025-ecs-cluster.

### Створіть визначення задачі (AWS CloudShell):

nano task-definition.json
{
  "family": "django-app-task",
  "networkMode": "awsvpc",
  "containerDefinitions": [
    {
      "name": "fp-0720225-app",
      "image": "381492090902.dkr.ecr.eu-central-1.amazonaws.com/django-app:latest",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 8000,
          "hostPort": 8000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {"name": "DJANGO_SECRET_KEY", "value": "EJTNceXxdZ5jIe36AsTtkFRDybvT9ynBhtx_6VLgbIAYAHn1Fc98Aoj-2EmFU8s"},
        {"name": "DEBUG", "value": "False"},
        {"name": "DJANGO_LOGLEVEL", "value": "info"},
        {"name": "DJANGO_ALLOWED_HOSTS", "value": "internal-fp-072025-alb-1020846421.eu-central-1.elb.amazonaws.com,fp-072025-alb-public-949119113.eu-central-1.elb.amazonaws.com,localhost,127.0.0.1,*"},
        {"name": "DATABASE_ENGINE", "value": "postgresql"},
        {"name": "DATABASE_NAME", "value": "django_db"},
        {"name": "DATABASE_USERNAME", "value": "postgres"},
        {"name": "DATABASE_PASSWORD", "value": "127devopsql5432"},
        {"name": "DATABASE_HOST", "value": "django-db.c9ag0ssuo6fe.eu-central-1.rds.amazonaws.com"},
        {"name": "DATABASE_PORT", "value": "5432"}
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/django-app",
          "awslogs-region": "eu-central-1",
          "awslogs-stream-prefix": "django"
        }
      }
    }
  ],
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::381492090902:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::381492090902:role/ecsTaskExecutionRole"
}

Зареєструйте:
aws ecs register-task-definition --cli-input-json file://task-definition.json --region eu-central-1

### Створіть сервіс ECS (AWS Console):ECS > Clusters > fp-072025-ecs-cluster > Services > Create.
Сервіс: fp-task-definition-service-latest, Задача: django-app-task:13, Кількість задач: 1.
Subnets: subnet-06ee8e37242fb0c1b, subnet-0426f53d4b9ada204.
Security Group: sg-07044a8e5cfe6b17c.
Load Balancer: fp-072025-alb-public, Target Group: fp-072025-target-alb-ip-public.

### Налаштування ALB (AWS Console):EC2 > Load Balancers > fp-072025-alb-public > Listeners.
Додайте слухача (listener):
Протокол: HTTP, Порт: 80, Дія: Forward to fp-072025-target-alb-ip-public.

Security Group: sg-01ed10b93e2973501, Дозволити: HTTP 80 від 0.0.0.0/0.

### Налаштування RDS (AWS Console):
RDS > Databases > django-db.
Переконайтеся, що:
Ендпоінт: django-db.c9ag0ssuo6fe.eu-central-1.rds.amazonaws.com.
Security Group: sg-08aa7aec99a829cf2, Дозволити: TCP 5432 від sg-07044a8e5cfe6b17c (ECS) та 10.0.6.72 (EC2 bastion).

### Виконайте міграції через EC2 bastion (Session Manager):

sudo yum install python3 python3-pip git
pip3 install django psycopg2-binary
git clone https://github.com/dm-zhuk/django-app-aws-cloud.git
cd django-app-aws-cloud/django_app
export DJANGO_SECRET_KEY="EJTNceXxdZ5jIe36AsTtkFRDybvT9ynBhtx_6VLgbIAYAHn1Fc98Aoj-2EmFU8s"
export DATABASE_NAME="django_db"
export DATABASE_USERNAME="postgres"
export DATABASE_PASSWORD="127devopsql5432"
export DATABASE_HOST="django-db.c9ag0ssuo6fe.eu-central-1.rds.amazonaws.com"
export DATABASE_PORT="5432"
python3 manage.py migrate

### Наповнення бази даних (EC2 Session Manager):
У тій же директорії:

python3 manage.py shell

from polls.models import Question, Choice
from django.utils import timezone
q = Question.objects.create(question_text="What's your favorite color?", pub_date=timezone.now())
q.choice_set.create(choice_text="Blue", votes=0)
q.choice_set.create(choice_text="Red", votes=0)
q.choice_set.create(choice_text="Green", votes=0)
q.save()
exit()

### Створіть суперкористувача:

python3 manage.py createsuperuser

Ім’я: admin, Пароль: 127devopsql5432.

### Налаштування CloudWatch (AWS Console):
CloudWatch > Log groups > Переконайтеся, що /ecs/django-app існує.
Створіть Alarms: CloudWatch > Alarms > Create alarm.
Метрика: AWS/ECS, CPUUtilization, fp-072025-ecs-cluster, fp-task-definition-service-latest.
Умови: Static, Greater than 80%, 3/3 datapoints, 5-minute period.
Сповіщення: PollsAppAlerts (SNS topic).

### Перевірка деплою (Browser):
Відкрийте http://fp-072025-alb-public-949119113.eu-central-1.elb.amazonaws.com.
> Сторінка входу /admin/ з оформленням.
Перейдіть до /polls/, перевірте список опитувань.

### Налаштування автомасштабування та балансування
Автомасштабування та балансування навантаження забезпечують високу доступність і ефективність Django polls додатку через ALB та ECS Fargate.
ALB Конфігурація:
EC2 > Load Balancers > fp-072025-alb-public.
Слухачі:
HTTP:80, Forward to fp-072025-target-alb-ip-public.
Target Group:
fp-072025-target-alb-ip-public, Порт: 8000, Протокол: HTTP.
Health Check:
Шлях /health/, Код 200, Інтервал: 30 секунд, Поріг: 3.
Security Group:
sg-01ed10b93e2973501, Дозволити HTTP 80 від 0.0.0.0/0.

Причина вибору: ALB розподіляє трафік між задачами ECS Fargate, забезпечуючи високу доступність. Перевірки стану автоматично перенаправляють трафік до здорових (Healthy) задач, підтримуючи стабільність додатку.

### Налаштування автомасштабування ECS (AWS Console):
Перевірка поточного стану:
ECS > Clusters > fp-072025-ecs-cluster > Services > fp-task-definition-service-latest.
Поточна кількість задач: 1, CPU: 256 одиниць, Пам’ять: 512 МБ.

### Створення (Auto Scaling) політики автомасштабування:
ECS > Clusters > fp-072025-ecs-cluster > Services > fp-task-definition-service-latest > Update Service.
Увімкніть “Service Auto Scaling”:
Мінімальна кількість задач: 1. | Максимальна кількість задач: 4.

Додайте політику масштабування:
Тип: Target Tracking Scaling Policy.
Назва: PollsAppCPUScaling.
Метрика: ECSServiceAverageCPUUtilization.
Цільове значення: 70%.
Час розгортання: 300.
Час згортання: 300.

Причина вибору: Автомасштабування ECS на основі ECSServiceAverageCPUUtilization дозволяє динамічно додавати або видаляти задачі залежно від навантаження. Ціль 70% CPU забезпечує баланс між продуктивністю та вартістю, дозволяючи масштабувати до 4 задач при високому навантаженні та зменшувати до 1 при низькому.
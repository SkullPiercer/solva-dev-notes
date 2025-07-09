# Лекция: CI/CD (Continuous Integration / Continuous Delivery)

CI/CD (непрерывная интеграция и доставка/развёртывание) — это совокупность практик и подходов, направленных на автоматизацию процессов разработки, тестирования и доставки программного обеспечения. Это ключевой компонент современной DevOps-культуры.



## Что такое CI/CD?

- **CI (Continuous Integration)** — процесс, при котором разработчики часто интегрируют изменения в общий репозиторий, и каждый коммит автоматически проверяется через сборку и тесты.
- **CD (Continuous Delivery / Deployment)**:
  - **Delivery** — изменения автоматически проходят тесты и готовы к развёртыванию (но требуют ручного запуска деплоя).
  - **Deployment** — автоматическое развёртывание в production после прохождения всех этапов.



## Зачем нужен CI/CD?

- **Быстрая доставка новых фич**
- **Автоматическое тестирование и меньше багов**
- **Повторяемость и надёжность развёртывания**
- **Частые релизы без стресса**
- **Улучшенная командная работа и прозрачность процессов**



## Основные этапы CI/CD пайплайна

1. **Code** — разработчик вносит изменения
2. **Build** — сборка приложения (например, `Docker`, `npm build`, `javac`, и т.д.)
3. **Test** — юнит-тесты, интеграционные тесты, линтинг, статический анализ
4. **Package** — упаковка приложения (например, Docker image, JAR, дистрибутив)
5. **Release** — выкладка артефактов в реестр/репозиторий (например, Docker Registry, PyPI)
6. **Deploy** — развёртывание в окружении (staging, production)
7. **Monitor** — сбор метрик и логов, алерты, трассировка



## Популярные инструменты CI/CD

| Категория     | Инструменты                          |
|---------------|--------------------------------------|
| CI-сервер     | GitHub Actions, GitLab CI/CD, Jenkins, CircleCI, Travis CI |
| Сборка        | Docker, Make, Gradle, Maven          |
| Тестирование  | Pytest, JUnit, Selenium, Cypress     |
| Деплой        | Helm, ArgoCD, Ansible, Capistrano    |
| Мониторинг    | Prometheus, Grafana, ELK Stack       |



## Пример CI/CD пайплайна (GitHub Actions)

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run tests
        run: pytest

      - name: Build Docker image
        run: docker build -t myapp:latest .

      - name: Push image to registry
        run: |
          echo ${{ secrets.DOCKER_PASSWORD }} | docker login -u ${{ secrets.DOCKER_USERNAME }} --password-stdin
          docker tag myapp:latest myrepo/myapp:latest
          docker push myrepo/myapp:latest

      - name: Deploy to Kubernetes
        run: kubectl apply -f k8s/deployment.yaml
```



## Стратегии деплоя

- **Blue-Green Deployment** — два окружения: старое (blue) и новое (green); переключение происходит после успешной валидации
- **Canary Releases** — выкатываем изменения для части пользователей, наблюдаем за метриками
- **Rolling Updates** — постепенно заменяем инстансы без простоев
- **Feature Toggles** — управление фичами через флаги без деплоя



## Лучшие практики

- Поддерживайте пайплайн быстрым (до 10 мин)
- Используйте линтинг и форматирование кода (flake8, black, eslint)
- Разделяйте стадии по окружениям (dev → staging → prod)
- Используйте секреты безопасно (через vault, secrets manager, CI/CD secrets)
- Храните конфигурации отдельно от кода (например, Helm, dotenv)
- Добавьте уведомления в Slack/Telegram при сбоях



## CI/CD и инфраструктура как код

Часто используется в связке с:

- **Terraform / Pulumi** — для создания облачной инфраструктуры
- **Docker / Kubernetes** — как окружение для деплоя
- **Helm** — шаблоны конфигураций Kubernetes
- **Vault / AWS Secrets Manager** — безопасное управление секретами



## Проблемы и как их решать

| Проблема | Решение |
|---------|---------|
| Медленные пайплайны | Кэширование зависимостей, запуск в параллель |
| Непредсказуемый деплой | Чёткие шаги, логирование, откаты (rollback) |
| Утечка секретов | Используйте CI/CD secrets и не хардкодьте токены |
| Частые сбои | Покрытие тестами, анализ метрик, staging перед production |



## Заключение

CI/CD — это не просто автоматизация, это философия надёжной доставки. С правильно настроенным пайплайном вы можете:

- Релизить быстрее и безопаснее
- Улучшить качество кода
- Сократить количество ошибок
- Повысить командную продуктивность

Автоматизируйте всё, что можно автоматизировать. Начните с малого, но стройте пайплайн с прицелом на масштабируемость и безопасность.

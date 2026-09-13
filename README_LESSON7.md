# Урок 7: Розгортання Kubernetes — Telco Churn MLOps

## Середовище
- **Кластер**: k3s v1.36.4+k3s1 (single-node), запущений всередині WSL2 Ubuntu на Windows
- **Container runtime**: containerd (через k3s), образи зібрані за допомогою Docker Engine у WSL
- **Namespace**: `mlops`

## Архітектура
Три FastAPI мікросервіси, розгорнуті через Kustomize:
- **ML Predictor** (`churn-predictor:latest`) — модель прогнозування відтоку клієнтів, порт 8000, 2→4 репліки + HPA
- **RAG Service** (`churn-rag:latest`) — LangChain + ChromaDB для пошуку по документах, порт 8001, зберігання через PVC
- **Agent Service** (`churn-agent:latest`) — LLM-оркестратор, який звертається до ML та RAG сервісів

## Виконані кроки розгортання
1. Зібрано всі 3 Docker образи локально та імпортовано їх у containerd k3s (`docker save | k3s ctr images import -`)
2. Створено namespace, ConfigMaps (ml-config, agent-config, rag-config) та Secret (llm-secrets) через Kustomize
3. Розгорнуто через `kubectl apply -k k8s/k8s/base/`
4. Перевірено, що всі 4 поди в статусі Running (2x ML, 1x RAG, 1x Agent)
5. Перевірено service discovery: Agent коректно резолвить `http://churn-ml-svc:80` та `http://churn-rag-svc:80` всередині кластера
6. Протестовано ML ендпоінт `/predict` — повернув коректний прогноз відтоку
7. RAG сервіс функціонує (health OK); індексація документів вимагає справжнього OpenAI API ключа (протестовано з тестовим ключем — коректно повернуло помилку 401 замість краху, що підтверджує правильну роботу Secret)

## Практичні лабораторні роботи

### Лаба 1: Масштабування (Scaling)
- У деплойменті активний HorizontalPodAutoscaler (min=2, max=10, цілі CPU 70% / memory 80%)
- Вручну відмасштабовано `churn-ml-service` з 2 до 4 реплік через `kubectl scale`
- Перевірено: `READY 4/4`, всі поди Running

### Лаба 2: Rolling Update (оновлення без простою)
- Rolling update запущено через зміну змінної середовища `MODEL_VERSION` (1.0.0 → 1.1.0) командою `kubectl set env`
- Спостерігали за `kubectl rollout status` — оновлення завершилось без простою (старі поди видалялись лише після готовності нових)
- Перевірено нову версію через ендпоінт `/`
- Підтверджено 2 ревізії в `kubectl rollout history`

### Лаба 3: Відмовостійкість / Самовідновлення
- Вручну видалено запущений ML под
- Kubernetes автоматично створив новий под протягом 23 секунд
- Загальна кількість реплік залишалась 4 протягом усього процесу, без ручного втручання

## Відомі обмеження
- Ingress використовує nginx-специфічні анотації; k3s за замовчуванням має Traefik, тому маршрутизацію через Ingress не тестували в цьому запуску (замість цього використано `kubectl port-forward`)
- Функціонал RAG/Agent з реальним LLM потребує справжнього OpenAI API ключа (у цьому запуску використано тестовий ключ-заглушку)

## Як відтворити
\`\`\`bash
cd k8s/k8s/base
kubectl apply -k .
kubectl get pods -n mlops
\`\`\`

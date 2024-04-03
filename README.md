# Вітаю усіх у репозиторії тестового додатку **"spam2000"**

Для розгортання я використовував наступні інструменти:

- Розгортання через Docker Desktop Kubernetes cluster.
![Kubernetes](/img/kubernetes.jpg)

Також необхідно встановити залежності:
- Встановити Helm:
  ```bash
  brew install helm
  ```
- Встановити kubectl:
  ```bash
  brew install kubectl
  ```

Далі створити chart директорію:
```bash
helm create spam2000
```

Основна архітектура складається з:
```
spam2000/
├── charts/
├── templates/
│   ├── spam2000-deployment.yaml
│   ├── prometheus-deployment.yaml
│   ├── grafana-deployment.yaml
├── Chart.yaml
└── values.yaml
```

Для початку створимо namespace, де буде запущено наш додаток і моніторинг:
```bash
kubectl create namespace spam2000
```

### Конфігурації

У файлі `values.yaml` вказані наступні параметри:

- **spam2000**: Опис додатку з іменем, докер-образом і версією, типом сервісу (NodePort) і портом, а також хелсчеками.

- **Prometheus**: Конфігурація для Prometheus включає ім'я, образ і версію, ConfigMap name, (PVC) об'ємом в 10Gi та сервіс типу NodePort із вказаним портом.

- **Grafana**: Вказує на налаштування Grafana, включаючи ім'я, докер-образ з тегом, ConfigMap name, PVC на 10Gi і NodePort сервіс із вказаним портом для доступу.


### Основна конфігурація

- `grafana-deployment.yaml`:
  - PVC мапить на хост директорію `/var/lib/grafana` і виділяє 10 GB.
  - ConfigMap з конфігурацією для автоматичного додавання Prometheus як DataSource.
  - Deployment та Service з NodePort (для продакшену використовуємо Ingress).

- `prometheus-deployment.yaml`:
  - PVC мапить на хост директорію `/prometheus` і виділяє 10 GB.
  - ConfigMap з конфігурацією для автоматичного скейпінгу нашого додатку кожні 15 секунд.
  - Основний Deployment Prometheus та Service з NodePort (для продакшену використовуємо Ingress).

- `spam2000-deployment.yaml`:
  - Основний Deployment з Health Checks.
  - Service з NodePort (для продакшену використовуємо Ingress).

Після підготовки конфігурацій, я встановив наш додаток, вказуючи namespace:
```bash
helm install spam2000 ./spam2000 --namespace spam2000
```

Перевіримо, чи всі компоненти запущені та працюють:
```bash
kubectl get all -n spam2000
```
![kubectl](/img/kubectl_get.jpg)

### Доступ до додатків

- SPAM2000:
![spam2000](/img/spam2000.jpg)

- Prometheus:
![prometheus](/img/prometheus.jpg)

- Grafana:
![grafana](/img/grafana.jpg)

## Налаштування Grafana

Після встановлення додатку, наступним кроком є налаштування Grafana. У репозиторії ви знайдете вже готовий дашборд `grafana.json`, який легко імпортувати та використовувати.

### Вхід у систему

1. **Залогіньтесь** в Grafana.
2. **Засетайте пароль** для вашого акаунта.

### Створення блоків на дашборді

#### Топ 5 користувачів за кількістю замовлень

- **Тип**: Pie Chart
- **PromQL**: `topk(5, sum by (email) (ordered_histogram_count))`
- **Налаштування**:
  - Transparent background: enable
  - Legend values: value
- **Пояснення**:
  Цей запит використовує функцію `topk` для відображення топ-5 користувачів за кількістю замовлень, сумованих за електронною адресою. За допомогою `sum by (email)` ми агрегуємо `ordered_histogram_count` для кожного користувача, а `topk(5, ...)` визначає п'ять користувачів з найвищою сумою.
> **Примітка**: Можливо, графік покаже більше ніж 5 користувачів, оскільки функція `topk` відображає всіх користувачів з однаковими значеннями.

#### Середній час обробки замовлення

- **Тип**: Time Series
- **PromQL**: `sum(ordered_histogram_sum) / 1000 / sum(ordered_histogram_count)`
- **Налаштування**:
  - Transparent background: enable
  - Legend vis: disable
  - Unit: Time(seconds)
- **Пояснення**:
    Цей блок вимірює середній час обробки замовлення. Я розраховую його, використовуючи функцію `sum(ordered_histogram_sum) / 1000 / sum(ordered_histogram_count)`, де я вираховую загальну кількість часу, витраченого на замовлення. Це число я ділю на 1000, щоб перевести результат у секунди, та далі ділю на загальну кількість замовлень. Таким чином, ми можемо отримати наступний результат:
- **Середній час** = \( \frac{13089.354662205983}{1000} \div 2000 \) = 0.0065 (час обробки на одне замовлення)


#### Загальна кількість оброблених замовлень за останні 5 хвилин

- **Тип**: Stat
- **PromQL:** `sum(ordered_histogram_count) - sum(ordered_histogram_count offset 5m)`
- **Налаштування**:
  - Calculation: Count
  - Graph Mode: None
  - Transparent Background: Enable
- **Пояснення**:
  Цей блок вираховує загальну кількість оброблених замовлень за останні 5 хвилин. Використовуються наступні метрики:
  - `sum(ordered_histogram_count)`: Обчислює загальну кількість замовлень, що відбулися "зараз", сумуючи значення `ordered_histogram_count` по всіх даних.
  - `sum(ordered_histogram_count offset 5m)`: Обчислює загальну кількість замовлень, що відбулися 5 хвилин тому, сумуючи значення `ordered_histogram_count`, взяті з затримкою у 5 хвилин. Це дозволяє порівняти поточну активність з активністю у минулому.

#### Інформація про співробітників

- **Тип**: Table
- **PromQL:** `sum by (email, exported_job)(name_gauge)`
- **Налаштування:**
  - Legend: Verbose
  - Format: Table
- Transform:
  - Group by:
    - email: Group by
    - exported_job: Group by
  - Value - Calculate: Last
- Transparent Background: Enable
- **Пояснення**:
  Цей блок призначений для виведення загальної інформації про співробітників у формі таблиці. Для цього використовується метрика `name_gauge`, з якої виводиться інформація у вигляді: ім'я співробітника | електронна пошта | значення.

### Результат

Після налаштування всіх блоків, дашборд буде виглядати так:
![Dashboard](/img/dashboard.jpg)


### Подяка

Окремо хочу подякувати за це цікаве ТЗ , чесно кажучі random метрики мене повністюю вибили з розуміння що відбувається , тому я вирішив їх проігнорувати, та взяв більш зрозумілі метрики, сподіваюсь це не дуже вплине на мою роботу!:)


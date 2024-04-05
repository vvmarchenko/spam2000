# Вітаю усіх у репозиторії тестового додатку **"spam2000"**

Для розгортання я використовував наступні інструменти:

- Розгортання через minikube.

```bash
minikube start --driver=hyperkit
```
> **Примітка**: Ви можете використати будь який драйвер для мінікуб, але в мене він працює тільке через hyperkit. LOVE MAC INTEL <3

Також необхідно встановити залежності:
- Встановити Helm:
  ```bash
  brew install helm
  ```
- Встановити kubectl:
  ```bash
  brew install kubectl
  ```

Далі я створив chart директорію:
```bash
helm create spam2000
```

> **Примітка**: Ви можете просто зробити `git clone` мого репозиторію , з готовими конфігураціями.и

Основна архітектура складається з:
```
spam2000/
├── charts/
├── templates/
│   ├── grafana-dashboard-configmap.yaml
│   ├── servicemonitor.yaml
│   ├── spam2000-deployment.yaml
├── Chart.yaml
└── values.yaml
```


### Конфігурації

У файлі `values.yaml` описано наступні конфігурації та компоненти:

- `Global Labels`: Загальні мітки, які застосовуються на глобальному рівні.

- `spam2000`: Основні налаштування для додатка spam2000, включаючи інформацію про образ, сервіс, стратегію оновлення, реплікацію та перевірки життєздатності та готовності.

- `Kube-Prometheus-Stack`: Конфігурації для Prometheus і Grafana, включаючи налаштування специфікацій Prometheus, налаштування сервісу, конфігурації зберігання, а також додаткові параметри для Grafana, включно з PVC, типом сервісу та додатковими datasource.

- `Dashboard`: Зміні для нашого дашборд включаючи datasource, назву та uid.

Конфігурація `Chart.yaml` описує основні властивості для Helm чарту, також ми вказуємо dependencies `kube-prometheus-stack` - Чарт з репозиторію prometheus-community, версія "55.2.0". Ця залежність автоматично встановлює необхідні компоненти для моніторингу за допомогою Prometheus і Grafana в контексті додатку spam2000.



### Основна конфігурація

- `grafana-dashboard-configmap.yaml`:
  - Містить configmap з основним grafana-dashboard (використовуючи змінні helm) для нашого додатку spam2000. 

- `servicemonitor.yaml`:
  - Описує об'єкт ServiceMonitor для Prometheus який використовується для визначення, як і звідки збирати метрики нашого app.

- `spam2000-deployment.yaml`:
  - Основний Deployment з Health Checks.
  - Service з NodePort (для продакшену використовуємо Ingress).

### Після підготовки конфігурацій, перейдемо до pre-install нашого стеку:

1. **Додавання репозиторію Helm**:
   Для початку, потрібно додати репозиторій `prometheus-community`, що містить потрібні чарти для Prometheus:
   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   ```

2. **Оновлення репозиторію Helm**:
   Після додавання нового репозиторію, виконайте оновлення, щоб переконатися, що ви маєте останні версії чартів:
   ```bash
   helm repo update
   ```

3. **Оновлення залежностей чарту**:
   Перед встановленням, переконайтеся, що всі залежності чарту оновлені. Перемістіться до каталогу з нашим чартом (`spam2000`) та виконайте оновлення залежностей:
   ```bash
   cd spam2000 && helm dependency update
   ```
   Цей крок забезпечить завантаження та оновлення всіх необхідних залежностей для нашого чарту.

4. **Встановлення чарту**:
   Нарешті, встановіть чарт `spam2000`, створивши новий namespace `spam2000` для цього додатку:
   ```bash
   helm install spam2000 . --create-namespace -n spam2000
   ```

Перевіримо, чи всі компоненти запущені та працюють:
```bash
kubectl get all -n spam2000
```
![kubectl](/img/kubectl_get.jpg)

### Доступ до додатків

**Перейдемо до перевірки доступу до нашого стеку**:

- Використайте команду `minikube ip` для отримання айпі вашого кластера.

- SPAM2000:
`192.168.65.3:30923/metrics`
![spam2000](/img/spam2000.jpg)

- Prometheus:
`192.168.65.3:30090`
![prometheus](/img/prom.jpg)

- Grafana:
`192.168.65.3:30080`
![grafana](/img/grafana.jpg)

## Налаштування Grafana

Після встановлення додатку, наступним кроком є налаштування Grafana (зміна адмін пароля).
> **Примітка**: Наш дашборд імпортується автоматично з ConfigMap , та має назву вказану в values.yaml `spam2000`.

### Вхід у систему



1. **Щоб отримати пароль для Grafana, треба дістати його з kuber secret**:
```bash
kubectl get secret -n spam2000 spam2000-grafana -o jsonpath="{.data.admin-password}" | base64 --decode; echo
```

2. **Залогінитись в Grafana, та змінити admin pass**

> **Примітка**: Я намагався встановити пароль через `--set grafana.adminPassword="admin"`, але чомусь пароль не підтягуєтсья , на форумах вказували саме цю команду --set.

### Огляд Dashboard "spam2000"

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
- **Середній час** = ( 13089.354662205983/1000 ) / 2000 = 0.0065 (час обробки на одне замовлення)


#### Загальна кількість оброблених замовлень за останню годину

- **Тип**: Stat
- **PromQL:** `sum(increase(ordered_histogram_count[1h]))`
- **Налаштування**:
  - Calculation: Count
  - Graph Mode: None
  - Transparent Background: Enable
- **Пояснення**:
  Цей блок вираховує загальну кількість оброблених замовлень за останні 5 хвилин. Використовуються наступні метрики:
  - `increase(ordered_histogram_count[1h])`: Функція `increase` в PromQL використовується для обчислення зміни значення метрики протягом вказаного періоду часу (у цьому випадку, 1 годину). Це дозволяє визначити, наскільки зросла кількість замовлень, зареєстрованих у гістограмі `ordered_histogram_count`, за останню годину.
  - `sum(...)`: Оператор `sum` потім агрегує всі зміни, виявлені `increase`, по всіх інстансах чи лейблах метрики `ordered_histogram_count`, щоб отримати загальну кількість змін за останню годину.

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

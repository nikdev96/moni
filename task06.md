# Задание 6

## Вопрос

Какие из ниже перечисленных систем относятся к push модели, а какие к pull? А может есть гибридные?

- Prometheus
- TICK
- Zabbix
- VictoriaMetrics
- Nagios

## Ответ

### Классификация систем мониторинга

#### 1. **Prometheus** - Pull (с возможностью Push через Pushgateway)

**Основная модель:** Pull

**Как работает:**
- Prometheus Server периодически опрашивает (scrape) экспортеры и приложения
- Метрики экспонируются через HTTP endpoint (обычно `/metrics`)
- Service Discovery автоматически находит таргеты

**Push-компонент:**
- **Pushgateway** - промежуточный сервис для batch jobs и короткоживущих процессов
- Приложения пушат метрики в Pushgateway, а Prometheus их оттуда забирает

**Вывод:** Преимущественно Pull, гибридная через Pushgateway

```yaml
# Пример scrape конфигурации (Pull)
scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
    scrape_interval: 15s
```

---

#### 2. **TICK Stack** (Telegraf, InfluxDB, Chronograf, Kapacitor) - Push

**Основная модель:** Push

**Как работает:**
- **Telegraf** (агент) собирает метрики и отправляет их в InfluxDB
- Telegraf поддерживает 200+ input plugins
- Данные пушатся по протоколам HTTP, UDP, TCP

**Особенности:**
- Telegraf может работать и в режиме pull (опрашивать endpoint'ы)
- Но архитектура спроектирована под push-модель
- InfluxDB принимает данные через HTTP API

**Вывод:** Преимущественно Push

```toml
# Telegraf конфигурация (Push в InfluxDB)
[[outputs.influxdb]]
  urls = ["http://localhost:8086"]
  database = "telegraf"

[[inputs.cpu]]
  percpu = true
  totalcpu = true
```

---

#### 3. **Zabbix** - Гибридная (Pull + Push)

**Модель:** Гибридная (поддерживает оба режима)

**Pull режим (Passive checks):**
- Zabbix Server опрашивает Zabbix Agent
- Агент слушает порт 10050
- Сервер запрашивает конкретные метрики

**Push режим (Active checks):**
- Zabbix Agent сам отправляет данные на сервер
- Агент получает список метрик от сервера
- Агент периодически пушит собранные данные

**Дополнительно:**
- **Zabbix Trapper** - принимает данные от внешних скриптов (push)
- **Zabbix Sender** - утилита для отправки метрик (push)

**Вывод:** Гибридная система (полноценная поддержка обоих режимов)

```xml
<!-- Passive check (Pull) -->
<Item>
  <Type>Zabbix agent</Type>
  <Key>system.cpu.load</Key>
</Item>

<!-- Active check (Push) -->
<Item>
  <Type>Zabbix agent (active)</Type>
  <Key>system.cpu.load</Key>
</Item>

<!-- Trapper (Push) -->
<Item>
  <Type>Zabbix trapper</Type>
  <Key>custom.metric</Key>
</Item>
```

```bash
# Отправка метрики через zabbix_sender (Push)
zabbix_sender -z zabbix-server -s "Web Server" -k custom.metric -o 100
```

---

#### 4. **VictoriaMetrics** - Гибридная (Pull + Push)

**Модель:** Гибридная

**Pull режим:**
- Полная совместимость с Prometheus
- Может использовать Prometheus scrape конфигурацию
- Опрашивает /metrics endpoints

**Push режим:**
- Нативная поддержка push через HTTP API
- **vmagent** может работать как в pull, так и в push режиме
- Поддержка push протоколов: InfluxDB line protocol, Graphite, OpenTSDB

**Особенности:**
- Более гибкая, чем Prometheus
- Может принимать данные напрямую без промежуточных компонентов
- Поддержка remote write protocol

**Вывод:** Гибридная система (полноценная поддержка обоих режимов)

```yaml
# Pull mode (Prometheus-совместимый)
scrape_configs:
  - job_name: 'my-app'
    static_configs:
      - targets: ['localhost:8080']

# Push mode через HTTP API
# curl -d 'metric_name{label="value"} 123' http://victoriametrics:8428/api/v1/import/prometheus
```

---

#### 5. **Nagios** - Pull

**Основная модель:** Pull

**Как работает:**
- Nagios Server выполняет проверки (checks) на удалённых хостах
- **Active checks:** Nagios сам опрашивает сервисы
- **NRPE** (Nagios Remote Plugin Executor) - Nagios инициирует проверку, агент её выполняет

**Pseudo-Push режим:**
- **NSCA** (Nagios Service Check Acceptor) - принимает пассивные проверки
- Но это скорее исключение для специальных случаев
- Основная архитектура - pull

**Вывод:** Преимущественно Pull

```cfg
# Active check (Pull) - Nagios опрашивает
define service {
    service_description     HTTP
    check_command           check_http
    check_interval          5
}

# NRPE check (Pull) - Nagios инициирует через NRPE
define service {
    service_description     CPU Load
    check_command           check_nrpe!check_load
}

# Passive check через NSCA (Pseudo-Push)
define service {
    service_description     Backup Status
    active_checks_enabled   0
    passive_checks_enabled  1
}
```

---

### Итоговая таблица

| Система | Модель | Основной режим | Дополнительные возможности |
|---------|--------|----------------|---------------------------|
| **Prometheus** | Гибридная | **Pull** | Push через Pushgateway |
| **TICK** | Преимущественно Push | **Push** | Telegraf может делать pull |
| **Zabbix** | **Гибридная** | Pull + Push | Passive/Active checks, Trapper |
| **VictoriaMetrics** | **Гибридная** | Pull + Push | Prometheus-совместимость + native push |
| **Nagios** | Преимущественно Pull | **Pull** | Passive checks через NSCA |

---

### Выводы по категориям

**Pull-системы:**
- Nagios (с минимальной поддержкой push)
- Prometheus (с поддержкой push через Pushgateway)

**Push-системы:**
- TICK Stack (Telegraf push'ит в InfluxDB)

**Гибридные системы (полноценная поддержка обоих режимов):**
- Zabbix (активные и пассивные проверки на равных)
- VictoriaMetrics (pull и push natively)

---

### Практические рекомендации

1. **Для классической инфраструктуры:** Prometheus (pull) или Zabbix (гибрид)
2. **Для cloud/kubernetes:** Prometheus или VictoriaMetrics (pull + service discovery)
3. **Для IoT/edge computing:** TICK (push) или Zabbix active checks
4. **Для mixed environments:** Zabbix или VictoriaMetrics (максимальная гибкость)
5. **Legacy monitoring:** Nagios (pull, проверенная годами)

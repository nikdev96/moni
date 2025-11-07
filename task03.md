# Задание 3

## Вопрос

Вашей DevOps команде в этом году не выделили финансирование на построение системы сбора логов. Разработчики в свою очередь хотят видеть все ошибки, которые выдают их приложения. Какое решение вы можете предпринять в этой ситуации, чтобы разработчики получали ошибки приложения?

## Ответ

### Решения для сбора логов БЕЗ финансирования:

#### 1. Использование Open Source решений:

**ELK Stack (самостоятельный деплой):**
- **Elasticsearch** - хранение и индексация логов
- **Logstash / Fluentd / Filebeat** - сбор и агрегация логов
- **Kibana** - визуализация и поиск
- **Стоимость:** Бесплатно (требуются только ресурсы сервера)
- **Минусы:** Требует ресурсов для развертывания и поддержки

**Loki + Grafana:**
- Легковесная альтернатива ELK
- Интегрируется с Prometheus
- Меньше требований к ресурсам
- **Стоимость:** Бесплатно
- **Плюсы:** Проще в настройке и эксплуатации чем ELK

**Graylog:**
- Open Source платформа для управления логами
- Проще в настройке чем ELK
- Хороший UI из коробки
- **Стоимость:** Бесплатно (Community Edition)

#### 2. Централизованное логирование на файловой системе:

**Простое решение с rsyslog:**
```bash
# Настройка rsyslog для централизованного сбора логов
# На сервере логов:
# /etc/rsyslog.conf
$ModLoad imudp
$UDPServerRun 514

$template RemoteLogs,"/var/log/remote/%HOSTNAME%/%PROGRAMNAME%.log"
*.* ?RemoteLogs

# На клиентах:
*.* @log-server:514
```

**Плюсы:**
- Встроено в Linux
- Нулевые затраты
- Простая настройка

**Минусы:**
- Нет удобного UI
- Поиск только через grep
- Нет долговременного хранения без дополнительных настроек

#### 3. Cloud Free Tiers (бесплатные уровни облачных сервисов):

**Datadog Free Tier:**
- 5 хостов бесплатно
- Хранение логов 3 дня
- Ограничение: 500MB логов в день

**New Relic Free Tier:**
- 100 GB данных в месяц бесплатно
- 1 бесплатный full platform user

**Grafana Cloud Free Tier:**
- 50 GB логов в месяц
- Хранение 14 дней

**Elastic Cloud Free Trial:**
- 14 дней бесплатно для тестирования

#### 4. Решения на основе контейнеров:

**Docker + Fluentd + Elasticsearch + Kibana:**
```yaml
# docker-compose.yml
version: '3'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - es-data:/usr/share/elasticsearch/data

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

  fluentd:
    build: ./fluentd
    volumes:
      - ./fluentd/conf:/fluentd/etc
      - /var/log:/var/log:ro
    ports:
      - "24224:24224"
    depends_on:
      - elasticsearch

volumes:
  es-data:
```

**Плюсы:**
- Быстрое развертывание
- Легко масштабировать
- Портативность

#### 5. Скрипты для парсинга и уведомлений:

**Python-скрипт для мониторинга логов и отправки в Slack/Telegram:**
```python
#!/usr/bin/env python3
import re
import time
import requests
from pathlib import Path

LOG_FILE = "/var/log/application/app.log"
WEBHOOK_URL = "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
ERROR_PATTERN = r"(ERROR|FATAL|Exception)"

def send_alert(message):
    payload = {"text": f"⚠️ Application Error:\n```{message}```"}
    requests.post(WEBHOOK_URL, json=payload)

def monitor_log():
    with open(LOG_FILE, 'r') as f:
        f.seek(0, 2)  # Go to end of file
        while True:
            line = f.readline()
            if not line:
                time.sleep(1)
                continue
            if re.search(ERROR_PATTERN, line):
                send_alert(line.strip())

if __name__ == "__main__":
    monitor_log()
```

Запуск как systemd service для непрерывного мониторинга.

#### 6. Использование журналирования Docker:

**Настройка Docker logging driver:**
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3",
    "labels": "production_status",
    "env": "os,customer"
  }
}
```

**Централизованный сбор с помощью docker logs:**
```bash
# Скрипт для сбора логов ошибок из всех контейнеров
#!/bin/bash
docker ps -q | while read container_id; do
    docker logs --since 5m $container_id 2>&1 | grep -i "error"
done
```

#### 7. Использование systemd journal:

**journalctl с централизованным хранением:**
```bash
# Настройка journald для персистентного хранения
# /etc/systemd/journald.conf
[Journal]
Storage=persistent
SystemMaxUse=500M

# Просмотр ошибок приложения
journalctl -u application.service -p err -f

# Экспорт в JSON для последующей обработки
journalctl -u application.service -o json > /var/log/app-errors.json
```

### Рекомендуемое решение:

**Для минимальных затрат и быстрого старта:**

1. **Короткий срок (Quick Win):**
   - Использовать **Grafana Cloud Free Tier** (50GB/месяц)
   - Или **Loki + Grafana** на одном сервере
   - Настроить алерты на ошибки через email/Slack/Telegram

2. **Средний срок:**
   - Развернуть **Loki + Grafana** в Docker
   - Настроить Promtail для сбора логов с приложений
   - Создать дашборды для разработчиков

3. **Долгосрочное решение:**
   - Поднять полноценный **ELK/EFK stack**
   - Или использовать **Graylog** (проще в эксплуатации)
   - Настроить retention policy для оптимизации хранения

### Пример конфигурации Loki (рекомендуемое решение):

**docker-compose.yml:**
```yaml
version: "3"

services:
  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    volumes:
      - ./loki-config.yaml:/etc/loki/local-config.yaml
      - loki-data:/loki
    command: -config.file=/etc/loki/local-config.yaml

  promtail:
    image: grafana/promtail:latest
    volumes:
      - /var/log:/var/log:ro
      - ./promtail-config.yaml:/etc/promtail/config.yaml
    command: -config.file=/etc/promtail/config.yaml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
  loki-data:
  grafana-data:
```

### Итого:

**Без бюджета можно:**
- Использовать Open Source решения (Loki, Graylog, ELK)
- Использовать Free Tier облачных сервисов
- Написать простые скрипты для парсинга и алертинга
- Использовать встроенные возможности Linux (journald, rsyslog)

**Рекомендация:** Начать с **Loki + Grafana + Promtail** - это самое простое и эффективное решение для старта с нулевым бюджетом.

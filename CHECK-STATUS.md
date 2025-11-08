# Проверка статуса TICK Stack

## Текущий статус

✅ **InfluxDB** - работает (порт 8086)
✅ **Chronograf** - работает (порт 8888)
✅ **Kapacitor** - работает (порт 9092)
⚠️ **Telegraf** - нужна проверка логов

## Проблема: Telegraf не пишет данные в InfluxDB

Выполните следующие команды для диагностики:

### 1. Проверить все контейнеры

```bash
sudo docker ps -a
```

Все 4 контейнера должны быть в статусе "Up":
- influxdb
- telegraf
- chronograf
- kapacitor

### 2. Проверить логи Telegraf

```bash
sudo docker logs telegraf
```

Ищите ошибки подключения к InfluxDB или другие проблемы.

### 3. Проверить логи InfluxDB

```bash
sudo docker logs influxdb
```

### 4. Если Telegraf не запущен или упал

```bash
# Перезапустить Telegraf
sudo docker compose restart telegraf

# Проверить логи после перезапуска
sudo docker logs -f telegraf
```

### 5. Если есть ошибка конфигурации

Возможные проблемы:

**A) Неправильный URL InfluxDB в telegraf.conf**

Должно быть:
```toml
[[outputs.influxdb]]
  urls = ["http://influxdb:8086"]
  database = "telegraf"
```

**B) Неправильные credentials**

В docker-compose.yml:
```yaml
environment:
  - INFLUXDB_USER=telegraf
  - INFLUXDB_USER_PASSWORD=telegraf123
```

В telegraf.conf:
```toml
  username = "telegraf"
  password = "telegraf123"
```

### 6. Проверить данные в InfluxDB вручную

```bash
# Войти в контейнер InfluxDB
sudo docker exec -it influxdb influx

# В консоли influx:
> SHOW DATABASES
> USE telegraf
> SHOW MEASUREMENTS
> SELECT * FROM cpu LIMIT 5
> exit
```

### 7. Проверить что Telegraf имеет доступ к Docker socket

```bash
sudo docker exec telegraf ls -la /var/run/docker.sock
```

Должно быть что-то вроде:
```
srw-rw---- 1 root XXX 0 ... /var/run/docker.sock
```

### 8. Полный перезапуск (если ничего не помогло)

```bash
cd /home/nikita/monitoring/my-homework

# Остановить все
sudo docker compose down

# Запустить заново
sudo docker compose up -d

# Смотреть логи в реальном времени
sudo docker compose logs -f
```

## После исправления

Подождите 10-30 секунд после запуска Telegraf, затем проверьте:

```bash
# Проверка через curl
curl -s "http://localhost:8086/query?db=telegraf&q=SHOW+MEASUREMENTS"

# Должны появиться measurements: cpu, disk, mem, docker_container_cpu и др.
```

## Открыть Chronograf

После того как данные появятся в InfluxDB:

1. Откройте браузер: **http://localhost:8888**
2. Если требуется настройка:
   - InfluxDB URL: `http://influxdb:8086`
   - Username: `admin`
   - Password: `admin123`
3. Перейдите в раздел **Explore**

---

## Команды для быстрой диагностики

```bash
# Все в одном
cd /home/nikita/monitoring/my-homework
echo "=== Docker containers ===" && \
sudo docker ps && \
echo -e "\n=== Telegraf logs (last 20 lines) ===" && \
sudo docker logs --tail 20 telegraf && \
echo -e "\n=== InfluxDB measurements ===" && \
curl -s "http://localhost:8086/query?db=telegraf&q=SHOW+MEASUREMENTS"
```

Скопируйте результат и дайте мне знать, я помогу с диагностикой!

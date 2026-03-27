# Filebeat — агент сбора логов

Сбор системных логов и логов Docker-контейнеров с отправкой в Kafka по SSL.

```
git clone https://github.com/lRAYNl/filebeat.git ~/filebeat
```

---

## Содержание

- [Описание](#описание)
- [Требования](#требования)
- [Шаг 1 — Установка Filebeat](#шаг-1--установка-filebeat)
- [Шаг 2 — Получение и установка сертификатов](#шаг-2--получение-и-установка-сертификатов)
- [Шаг 3 — Настройка конфигурации](#шаг-3--настройка-конфигурации)
- [Шаг 4 — Запуск и проверка](#шаг-4--запуск-и-проверка)
- [Структура конфига](#структура-конфига)
- [Полезные команды](#полезные-команды)
- [Устранение неполадок](#устранение-неполадок)

---

## Описание

Filebeat читает:
- `/var/log/syslog` и `/var/log/auth.log` — системные логи
- `/var/lib/docker/containers/*/*.log` — логи всех Docker-контейнеров

И отправляет события в топик `kafka-logs` Kafka-кластера по SSL (порт **9092**).

---

## Требования

- Ubuntu 22.04 / Debian 12
- Filebeat 9.x
- Доступность Kafka-брокеров по порту `9092`
- Сертификаты: `ca.crt` (Kafka CA) — получить у администратора Kafka-кластера

---

## Шаг 1 — Установка Filebeat

```bash
# Импортировать GPG-ключ Elastic
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch \
  | sudo gpg --dearmor \
  -o /usr/share/keyrings/elasticsearch-keyring.gpg

# Добавить репозиторий
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] \
  https://artifacts.elastic.co/packages/9.x/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/elastic-9.x.list

# Установить
sudo apt-get update && sudo apt-get install filebeat -y

# Проверить версию
filebeat version
```

---

## Шаг 2 — Получение и установка сертификатов

Filebeat нужен только **`ca.crt` от Kafka-кластера** (не от ELK).  
Сертификат предоставляется с **kafka-node-01**.

### 2.1 Раздать ca.crt с kafka-node-01

**На kafka-node-01** — запустить HTTP-сервер:

```bash
cd ~/kafka/certs
python3 -m http.server 8888
```

**На хосте с Filebeat** — скачать и установить:

```bash
sudo mkdir -p /etc/filebeat/certs

sudo wget http://<IP_KAFKA_NODE_01>:8888/ca.crt \
  -O /etc/filebeat/certs/ca.crt

sudo chmod 644 /etc/filebeat/certs/ca.crt
```

Остановить сервер на kafka-node-01 (`Ctrl+C`).

Итоговая структура:

```
/etc/filebeat/certs/
└── ca.crt    # корневой CA Kafka (для проверки сертификата брокера)
```

---

## Шаг 3 — Настройка конфигурации

```bash
# Скопировать конфиг из репозитория
sudo cp ~/filebeat/filebeat.yml /etc/filebeat/filebeat.yml

# Открыть для редактирования
sudo nano /etc/filebeat/filebeat.yml
```

В секции `output.kafka` указать актуальные IP-адреса Kafka-брокеров:

```yaml
output.kafka:
  hosts: ["<KAFKA_IP_1>:9092", "<KAFKA_IP_2>:9092", "<KAFKA_IP_3>:9092"]
  topic: "kafka-logs"
  ssl:
    enabled: true
    certificate_authorities: ["/etc/filebeat/certs/ca.crt"]
    verification_mode: none
```

---

## Шаг 4 — Запуск и проверка

```bash
# Проверить конфигурацию
sudo filebeat test config -c /etc/filebeat/filebeat.yml

# Проверить подключение к Kafka
sudo filebeat test output -c /etc/filebeat/filebeat.yml

# Включить автозапуск и стартовать
sudo systemctl enable filebeat
sudo systemctl start filebeat

# Проверить статус
sudo systemctl status filebeat
```

---

## Структура конфига

```yaml
filebeat.inputs:

  # Системные логи
  - type: filestream
    id: my-system-logs
    enabled: true
    paths:
      - /var/log/syslog
      - /var/log/auth.log
    tags: ["syslog"]

  # Логи Docker-контейнеров
  - type: filestream
    id: my-docker-logs
    enabled: true
    paths:
      - /var/lib/docker/containers/*/*.log
    parsers:
      - container:
          stream: all
    tags: ["docker"]

output.kafka:
  hosts: ["<KAFKA_IP_1>:9092", "<KAFKA_IP_2>:9092", "<KAFKA_IP_3>:9092"]
  topic: "kafka-logs"
  ssl:
    enabled: true
    certificate_authorities: ["/etc/filebeat/certs/ca.crt"]
    verification_mode: none
```

---

## Полезные команды

```bash
# Логи в реальном времени
sudo journalctl -u filebeat -f

# Перезапуск после изменения конфига
sudo systemctl restart filebeat

# Остановка
sudo systemctl stop filebeat

# Ручной тест с подробным выводом
sudo filebeat -e -c /etc/filebeat/filebeat.yml -d "kafka"
```

---

## Устранение неполадок

| Симптом | Причина | Решение |
|---------|---------|---------|
| `SSL handshake failed` | Неверный `ca.crt` (взят от ELK, а не от Kafka) | Использовать `ca.crt` с **kafka-node-01**, не с elk-node-01 |
| `connection refused` на порту 9092 | Kafka недоступна или порт закрыт | Проверить IP-адреса в `hosts`, доступность порта 9092 |
| `permission denied` на `/var/lib/docker` | Нет прав на чтение docker-логов | `sudo usermod -aG docker filebeat` или запустить от root |
| Нет данных в ES-индексе `filebeat-*` | Логи не поступают в Kafka | Проверить `filebeat test output`, логи Logstash |
| `open /var/log/syslog: no such file` | На системе нет syslog | Убрать путь из конфига или `sudo apt install rsyslog` |

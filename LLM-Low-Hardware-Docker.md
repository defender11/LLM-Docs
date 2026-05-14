# Ollama в Docker без глобальной установки

Инструкция для случая, когда на сервере уже есть Docker, а ставить Ollama напрямую в систему не хочется.

Ollama официально предоставляет Docker-образ `ollama/ollama`; базовый CPU-only запуск выглядит так: `docker run -d -v ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama`. :contentReference[oaicite:0]{index=0}

---

# 1. Проверка Docker

```bash
docker --version
```

```bash
docker compose version
```

Если `docker compose` не работает, можно использовать обычный `docker run`.

---

# 2. Проверка железа

```bash
uname -a
lscpu
cat /proc/meminfo
```

Особенно важно в `lscpu`:

```text
Flags:
```

Если нет:

```text
avx
avx2
fma
```

то на очень старых CPU, например Intel Atom D525, Ollama может не запуститься или упасть с `illegal instruction`.

Для таких машин надежнее `llama.cpp`, но Docker-вариант с Ollama всё равно можно попробовать.

---

# 3. Промт для подбора модели

```text
У меня есть Linux сервер.

Вот характеристики:

<вставить вывод команд>

uname -a
lscpu
cat /proc/meminfo

Подбери для него:
- подойдет ли Ollama в Docker
- какие модели Ollama он потянет
- какие модели лучше не запускать
- сколько RAM нужно
- сколько CPU threads использовать
- какой контекст ставить
- лучше использовать Ollama или llama.cpp
```

---

# 4. Рекомендации по моделям

## Очень слабый сервер

Например:

```text
Intel Atom D525
2 ядра / 4 потока
3 GB RAM
без AVX / AVX2
```

Можно пробовать:

| Модель | Команда | Комментарий |
|---|---|---|
| SmolLM 135M | `ollama run smollm:135m` | Самый безопасный старт |
| SmolLM 360M | `ollama run smollm:360m` | Чуть умнее, но медленнее |
| TinyLlama | `ollama run tinyllama` | Может быть тяжеловато |
| Qwen 0.5B | `ollama run qwen2.5:0.5b` | Если хватает RAM |

Не стоит запускать:

| Модель | Почему |
|---|---|
| llama3 / llama3.1 / llama3.2 7B+ | слишком тяжело |
| mistral | мало RAM |
| gemma 7B | мало RAM |
| qwen 7B+ | мало RAM |
| deepseek coder | слишком тяжело |

---

## 4-8 GB RAM

Можно пробовать:

```bash
ollama run smollm:360m
ollama run tinyllama
ollama run qwen2.5:0.5b
```

---

## 8-16 GB RAM

Можно пробовать:

```bash
ollama run qwen2.5:1.5b
ollama run gemma2:2b
ollama run phi3
```

---

## 16-32 GB RAM

Можно пробовать:

```bash
ollama run llama3.2
ollama run mistral
ollama run qwen2.5:7b
```

---

# 5. Вариант 1: запуск через docker run

## CPU-only запуск

```bash
docker run -d \
  --name ollama \
  --restart unless-stopped \
  -p 11434:11434 \
  -v ollama:/root/.ollama \
  ollama/ollama
```

Что здесь происходит:

| Параметр | Значение |
|---|---|
| `--name ollama` | имя контейнера |
| `--restart unless-stopped` | автозапуск после ребута |
| `-p 11434:11434` | проброс API порта |
| `-v ollama:/root/.ollama` | постоянное хранилище моделей |
| `ollama/ollama` | официальный Docker image |

---

# 6. Проверка контейнера

```bash
docker ps
```

Логи:

```bash
docker logs -f ollama
```

Проверка API локально:

```bash
curl http://127.0.0.1:11434/api/tags
```

Проверка API по сети:

```bash
curl http://SERVER_IP:11434/api/tags
```

---

# 7. Скачать модель внутрь Ollama

```bash
docker exec -it ollama ollama pull smollm:135m
```

Или сразу запустить:

```bash
docker exec -it ollama ollama run smollm:135m
```

Для слабого сервера начинать лучше именно с:

```bash
docker exec -it ollama ollama run smollm:135m
```

---

# 8. Использование локально

## Обычный Ollama API

```bash
curl http://127.0.0.1:11434/api/generate \
  -H "Content-Type: application/json" \
  -d '{
    "model": "smollm:135m",
    "prompt": "Hello. Answer shortly.",
    "stream": false
  }'
```

---

## OpenAI-compatible API

```bash
curl http://127.0.0.1:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "smollm:135m",
    "messages": [
      {
        "role": "user",
        "content": "Hello"
      }
    ],
    "max_tokens": 32
  }'
```

---

# 9. Использование по сети

Если сервер имеет IP:

```text
192.168.1.34
```

то проверка:

```bash
curl http://192.168.1.34:11434/api/tags
```

Запрос:

```bash
curl http://192.168.1.34:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "smollm:135m",
    "messages": [
      {
        "role": "user",
        "content": "Say hello in one short sentence"
      }
    ],
    "max_tokens": 32
  }'
```

---

# 10. Docker Compose вариант

Создать папку:

```bash
mkdir -p ~/docker/ollama
cd ~/docker/ollama
```

Создать файл:

```bash
nano docker-compose.yml
```

Содержимое:

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped

    ports:
      - "11434:11434"

    volumes:
      - ollama:/root/.ollama

    environment:
      - OLLAMA_HOST=0.0.0.0
      - OLLAMA_KEEP_ALIVE=5m
      - OLLAMA_NUM_PARALLEL=1
      - OLLAMA_MAX_LOADED_MODELS=1

volumes:
  ollama:
```

Запустить:

```bash
docker compose up -d
```

Проверить:

```bash
docker compose ps
docker compose logs -f
```

Остановить:

```bash
docker compose down
```

Остановить без удаления моделей:

```bash
docker compose down
```

Остановить и удалить volume с моделями:

```bash
docker compose down -v
```

---

# 11. Где лежат модели

В Docker volume:

```text
ollama
```

Посмотреть volume:

```bash
docker volume ls
```

Информация:

```bash
docker volume inspect ollama
```

Модели внутри контейнера лежат здесь:

```text
/root/.ollama
```

---

# 12. Ограничение ресурсов контейнера

Для слабого сервера можно ограничить CPU/RAM.

## Через docker run

```bash
docker run -d \
  --name ollama \
  --restart unless-stopped \
  -p 11434:11434 \
  -v ollama:/root/.ollama \
  --cpus="2" \
  --memory="2g" \
  ollama/ollama
```

## Через docker-compose.yml

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped

    ports:
      - "11434:11434"

    volumes:
      - ollama:/root/.ollama

    environment:
      - OLLAMA_HOST=0.0.0.0
      - OLLAMA_KEEP_ALIVE=5m
      - OLLAMA_NUM_PARALLEL=1
      - OLLAMA_MAX_LOADED_MODELS=1

    deploy:
      resources:
        limits:
          cpus: "2"
          memory: 2G

volumes:
  ollama:
```

Важно: секция `deploy.resources` полноценно работает в Docker Swarm, а в обычном `docker compose` может вести себя по-разному в зависимости от версии Docker Compose.

Для обычного сервера проще использовать `docker run --cpus --memory`.

---

# 13. Перезапуск

```bash
docker restart ollama
```

---

# 14. Остановка

```bash
docker stop ollama
```

---

# 15. Удаление контейнера без удаления моделей

```bash
docker rm ollama
```

Volume с моделями останется.

---

# 16. Полное удаление вместе с моделями

```bash
docker stop ollama
docker rm ollama
docker volume rm ollama
```

---

# 17. Обновление Ollama в Docker

Остановить и удалить старый контейнер:

```bash
docker stop ollama
docker rm ollama
```

Скачать свежий образ:

```bash
docker pull ollama/ollama:latest
```

Запустить заново:

```bash
docker run -d \
  --name ollama \
  --restart unless-stopped \
  -p 11434:11434 \
  -v ollama:/root/.ollama \
  ollama/ollama
```

Модели сохранятся, потому что они лежат в volume `ollama`.

---

# 18. PHP пример

```php
<?php

$ch = curl_init();

curl_setopt_array($ch, [
    CURLOPT_URL => 'http://SERVER_IP:11434/v1/chat/completions',
    CURLOPT_POST => true,
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER => [
        'Content-Type: application/json',
    ],
    CURLOPT_POSTFIELDS => json_encode([
        'model' => 'smollm:135m',
        'messages' => [
            [
                'role' => 'user',
                'content' => 'Hello',
            ],
        ],
        'max_tokens' => 64,
    ]),
]);

$response = curl_exec($ch);

if ($response === false) {
    echo curl_error($ch);
    curl_close($ch);
    exit;
}

curl_close($ch);

echo $response;
```

---

# 19. JavaScript пример

```javascript
const response = await fetch('http://SERVER_IP:11434/v1/chat/completions', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    model: 'smollm:135m',
    messages: [
      {
        role: 'user',
        content: 'Hello',
      },
    ],
    max_tokens: 64,
  }),
});

const data = await response.json();

console.log(data.choices[0].message.content);
```

---

# 20. Возможные проблемы

## 403 при скачивании install.sh

Это неважно, потому что Docker-вариант не использует `install.sh`.

---

## Контейнер запустился, но модель не отвечает

Проверить логи:

```bash
docker logs -f ollama
```

Проверить список моделей:

```bash
docker exec -it ollama ollama list
```

---

## Мало RAM

Симптомы:
- контейнер перезапускается
- запрос зависает
- сервер уходит в swap
- модель не грузится

Решение:
- взять модель меньше
- уменьшить количество параллельных запросов
- не держать много моделей в памяти

В compose:

```yaml
environment:
  - OLLAMA_NUM_PARALLEL=1
  - OLLAMA_MAX_LOADED_MODELS=1
  - OLLAMA_KEEP_ALIVE=1m
```

---

## Старый CPU без AVX

Симптомы:
- `illegal instruction`
- контейнер сразу падает
- в логах ошибки CPU-инструкций

Проверка:

```bash
lscpu | grep Flags
```

Если нет `avx`, лучше использовать `llama.cpp`, собранный с:

```bash
-DGGML_AVX=OFF
-DGGML_AVX2=OFF
-DGGML_FMA=OFF
```

---

# 21. Итоговая команда для слабого сервера

Минимальный вариант:

```bash
docker run -d \
  --name ollama \
  --restart unless-stopped \
  -p 11434:11434 \
  -v ollama:/root/.ollama \
  -e OLLAMA_HOST=0.0.0.0 \
  -e OLLAMA_NUM_PARALLEL=1 \
  -e OLLAMA_MAX_LOADED_MODELS=1 \
  -e OLLAMA_KEEP_ALIVE=1m \
  ollama/ollama
```

Скачать маленькую модель:

```bash
docker exec -it ollama ollama pull smollm:135m
```

Проверить:

```bash
curl http://127.0.0.1:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "smollm:135m",
    "messages": [
      {
        "role": "user",
        "content": "Hello"
      }
    ],
    "max_tokens": 32
  }'
```

---

# 22. Важное замечание

Docker решает проблему чистоты системы:

- не ставим Ollama глобально
- не засоряем `/usr/local/bin`
- модели лежат в Docker volume
- легко удалить контейнер
- легко обновить
- удобно переносить

Но Docker не решает проблему старого CPU.

Если CPU не поддерживает нужные инструкции, Ollama в Docker тоже может не заработать. В таком случае лучше использовать `llama.cpp` без Ollama.
# Локальный LLM сервер на слабом ПК (Intel Atom / 3 GB RAM)

Инструкция по запуску `llama.cpp` + локальной LLM на очень слабом железе.

Подходит для:
- старых mini-PC
- Atom
- Celeron
- древних офисных ПК
- тестовых серверов
- локального OpenAI-compatible API

---

# 1. Проверка железа

## CPU

```bash
lscpu
```

## RAM

```bash
cat /proc/meminfo
```

## Архитектура

```bash
uname -a
```

## Видеокарта

```bash
lspci | grep -i vga
```

---

# 2. Как понять какую модель потянет ПК

Особенно важно поле:

```text
Flags:
```

## Хорошо если есть:

- avx
- avx2
- fma

## Если НЕТ avx/avx2

(как у Intel Atom D525)

ТО:
- Ollama может не работать
- нужен `llama.cpp`
- нужны маленькие GGUF модели

---

# 3. Быстрый промт для ChatGPT

Можно просто скормить вывод:

```bash
uname -a
lscpu
cat /proc/meminfo
```

и использовать промт:

```text
У меня есть Linux сервер.

Вот характеристики:

<вставить вывод>

Подбери:
- какие LLM модели потянет
- какие GGUF квантовки использовать
- сколько контекста ставить
- сколько потоков CPU использовать
- подойдет ли ollama
- или лучше llama.cpp
```

---

# 4. Рекомендации по моделям

## Очень слабые ПК

### 2-4 GB RAM
### Старые CPU без AVX

Например:
- Intel Atom
- Core2Duo
- старые AMD

### Подходят:

| Модель | Размер | Скорость |
|---|---|---|
| SmolLM2 135M Q4 | ~100 MB | Быстро |
| SmolLM2 360M Q4 | ~250 MB | Норм |
| TinyLlama 1.1B Q2 | ~500 MB | Медленно |
| TinyLlama 1.1B Q4 | ~700 MB | Очень медленно |

### НЕ подходят:

- Llama 7B
- Mistral
- Gemma 7B
- Qwen 7B
- Deepseek

---

## Средние ПК

### 8-16 GB RAM
### AVX2 CPU

Подойдут:
- Gemma 2B
- Phi
- Qwen 2.5 3B
- TinyLlama Q8

---

## Нормальные ПК

### 16-32 GB RAM

Подойдут:
- 7B модели
- Mistral
- Llama3
- Qwen 7B

---

# 5. Почему llama.cpp вместо Ollama

Для старого железа лучше `llama.cpp` потому что:

- работает без AVX
- меньше overhead
- меньше RAM
- поддерживает древние CPU
- проще сборка
- OpenAI API уже встроен

---

# 6. Установка llama.cpp

## Установка зависимостей

```bash
sudo apt update

sudo apt install -y \
    git \
    build-essential \
    cmake
```

## Скачать llama.cpp

```bash
git clone https://github.com/ggml-org/llama.cpp

cd llama.cpp
```

---

# 7. Сборка для старого CPU

## Для CPU БЕЗ AVX

```bash
cmake -B build \
  -DGGML_AVX=OFF \
  -DGGML_AVX2=OFF \
  -DGGML_FMA=OFF \
  -DLLAMA_BUILD_TESTS=OFF \
  -DLLAMA_BUILD_EXAMPLES=OFF \
  -DLLAMA_BUILD_TOOLS=ON \
  -DCMAKE_BUILD_TYPE=Release
```

## Сборка

Для слабых ПК:

```bash
cmake --build build --target llama-cli llama-server -j1
```

Для чуть сильнее:

```bash
cmake --build build --target llama-cli llama-server -j2
```

---

# 8. Скачивание моделей

## Создать папку

```bash
mkdir models
```

## SmolLM2 135M

Скачать на основной ПК через браузер:

```text
https://huggingface.co/bartowski/SmolLM2-135M-Instruct-GGUF
```

Нужный файл:

```text
SmolLM2-135M-Instruct-Q4_K_M.gguf
```

## Передать на сервер

```bash
scp SmolLM2-135M-Instruct-Q4_K_M.gguf \
ubuntu@SERVER_IP:~/llama.cpp/models/
```

---

# 9. Локальный запуск

## Интерактивный чат

```bash
./build/bin/llama-cli \
  -m models/SmolLM2-135M-Instruct-Q4_K_M.gguf \
  -n 128 \
  -c 256
```

## Параметры

| Параметр | Значение |
|---|---|
| -m | путь к модели |
| -n | максимальный ответ |
| -c | размер контекста |

---

# 10. Запуск API сервера

## OpenAI-compatible API

```bash
./build/bin/llama-server \
  -m models/SmolLM2-135M-Instruct-Q4_K_M.gguf \
  --host 0.0.0.0 \
  --port 8080 \
  -c 128 \
  -t 2
```

---

# 11. Проверка работы

## Локально

```bash
curl http://127.0.0.1:8080/health
```

Ответ:

```json
{"status":"ok"}
```

## По сети

```bash
curl http://SERVER_IP:8080/health
```

---

# 12. Использование API

## Chat completion

```bash
curl http://SERVER_IP:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
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

# 13. PHP пример

```php
<?php

$ch = curl_init();

curl_setopt_array($ch, [
    CURLOPT_URL => 'http://SERVER_IP:8080/v1/chat/completions',
    CURLOPT_POST => true,
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER => [
        'Content-Type: application/json',
    ],
    CURLOPT_POSTFIELDS => json_encode([
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

curl_close($ch);

echo $response;
```

---

# 14. Автозапуск через systemd

## Создать service

```bash
sudo nano /etc/systemd/system/llama.service
```

## Конфиг

```ini
[Unit]
Description=llama.cpp server
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/llama.cpp

ExecStart=/home/ubuntu/llama.cpp/build/bin/llama-server \
    -m /home/ubuntu/llama.cpp/models/SmolLM2-135M-Instruct-Q4_K_M.gguf \
    --host 0.0.0.0 \
    --port 8080 \
    -c 128 \
    -t 2

Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

## Включить

```bash
sudo systemctl daemon-reload

sudo systemctl enable llama

sudo systemctl start llama
```

## Проверить

```bash
systemctl status llama
```

---

# 15. Производительность

## Intel Atom D525

### Результат:

| Параметр | Значение |
|---|---|
| Скорость prompt | ~2.4 tok/s |
| Скорость generation | ~2 tok/s |
| RAM | ~500-900 MB |
| Контекст | 128-256 |

---

# 16. Для чего подходит

## Хорошо подходит:

- тесты API
- локальный AI endpoint
- Telegram bot
- Home Assistant
- простые AI задачи
- автодополнение
- classification
- маленькие чат-боты

## Плохо подходит:

- кодогенерация
- reasoning
- большие модели
- длинный контекст
- embeddings
- RAG
- multi-agent

---

# 17. Полезные команды

## Остановить

```bash
Ctrl+C
```

## Проверить RAM

```bash
htop
```

## Проверить процесс

```bash
ps aux | grep llama
```

## Проверить порт

```bash
ss -tulpn | grep 8080
```

---

# 18. Итог

Даже древний:
- Intel Atom
- 2 ядра
- 3 GB RAM

может:
- запускать LLM
- держать OpenAI-compatible API
- работать как локальный AI сервер

Главное:
- использовать `llama.cpp`
- маленькие GGUF модели
- маленький context
- низкие квантовки Q2/Q4
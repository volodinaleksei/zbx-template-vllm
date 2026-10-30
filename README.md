# Zabbix шаблон: vLLM (LLM API)

Мониторинг [vLLM](https://github.com/vllm-project/vllm) OpenAI-совместимого сервера через Zabbix 7.x.

Шаблон собран для **Zabbix 7.4**, должен работать на 7.0+ (HTTP agent + dependent items + JS preprocessing).

## Как это работает

Три master-ITEM'а (тип `HTTP agent`, value `TEXT`, history отключён) забирают сырые ответы,
остальные ITEM'ы — **dependent** со JS-парсингом:

| Master | Эндпоинт | Интервал |
|---|---|---|
| `vllm.metrics.raw` | `http://{HOST.CONN}:{$VLLM.PORT}{$VLLM.METRICS.PATH}` (Prometheus) | 1m |
| `vllm.health.raw` | `/health` | 30s |
| `vllm.test.raw` | `{$VLLM.TEST.PATH}` (по умолчанию `/v1/models`) | 5m |

Метрики Prometheus парсятся регуляркой внутри JS-шага (Zabbix не умеет нативный Prometheus-парсинг
в HTTP agent без agent 2, а сервер может быть вообще без доступа к хосту).

## ITEM'ы

- `vllm.requests.running` / `vllm.requests.waiting` — запросы в обработке / в очереди (сумма по движкам)
- `vllm.kv_cache.usage` — занятость KV-cache, % (из `vllm:kv_cache_usage_perc` ×100)
- `vllm.ttft.avg` — среднее Time To First Token (`_sum` / `_count`)
- `vllm.preemptions` — преemption'ы (`vllm:num_preemptions_total`)
- `vllm.tokens.prompt` / `vllm.tokens.generation` — всего токенов (rate() на графике)
- `vllm.process.rss` — память процесса
- `vllm.health` / `vllm.test` — флаги 1/0 доступности эндпоинтов
- `vllm.models.count` — число моделей из `/v1/models`
- `net.tcp.service[http,,{$VLLM.PORT}]` — simple check порта

## LLD по моделям

Правило `vllm.model.discovery` (dependent от `/v1/models`) создаёт на каждую модель:
в обработке, в очереди, успешных запросов, prompt/generation токенов, размер контекста (`max_model_len`).

## Триггеры

| Событие | Sev | Условие |
|---|---|---|
| vLLM: сервер не отвечает (/health) | High | `/health` недоступен 5m |
| vLLM: порт {$VLLM.PORT} недоступен | High | TCP закрыт, но /health недавно отвечал (умер процесс, не сеть) |
| vLLM: API не отвечает (/health живой) | Medium | /health есть, но тестовый путь недоступен |
| vLLM: нет загруженных моделей | Warning | `/v1/models` вернул пустой список |
| vLLM: затор очереди | Warning | средняя очередь > `{$VLLM.QUEUE.MAX}` за 5m |
| vLLM: KV-cache > {$VLLM.KV.MAX}% | Warning | риск преemption'ов |
| vLLM: модель {#MODELID} выгружена | Warning | модель пропала из `/v1/models` |

## Макросы

| Макрос | По умолчанию | Назначение |
|---|---|---|
| `{$VLLM.PORT}` | `8000` | порт сервера |
| `{$VLLM.METRICS.PATH}` | `/metrics` | путь Prometheus-метрик |
| `{$VLLM.TEST.PATH}` | `/v1/models` | путь функциональной проверки |
| `{$VLLM.QUEUE.MAX}` | `5` | порог предупреждения: очередь |
| `{$VLLM.KV.MAX}` | `90` | порог предупреждения: KV-cache, % |

## Установка

1. Import `template_vllm.yaml` (Data collection → Templates → Import).
2. Создайте хост, interface: **Agent**, IP сервера vLLM (используется только как `{HOST.CONN}`;
   агент на хосте не нужен — все ITEM'ы HTTP/simple check).
3. Привяжите шаблон. При HTTPS/авторизации отредактируйте master-ITEM'ы вручную.

Проверено живым поллингом против vLLM с моделью `qwen3-vl-8b`.

## Grafana

Метрики на сервере стандартные Prometheus-имена (`vllm:*`) — если нужен дашборд,
Grafana может читать `/metrics` напрямую, этот шаблон её не заменяет, а дополняет алертингом.

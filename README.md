# Gemini API Proxy via Cloudflare Workers for OpenCode

Бесплатный прокси для использования Google Gemini в [OpenCode](https://opencode.ai) и любых других OpenAI-совместимых клиентах. Cloudflare Worker конвертирует OpenAI API формат (`/v1/chat/completions`) в Gemini API формат и обратно.

---

## Как это работает

```
OpenCode / любой OpenAI-клиент
        ↓
Cloudflare Worker (конвертер OpenAI ↔ Gemini)
        ↓
Google Gemini API (бесплатно)
```

---

## Шаг 1 — Получить Gemini API ключ

1. Зайди на [aistudio.google.com](https://aistudio.google.com)
2. Авторизуйся через Google-аккаунт
3. Нажми **Get API key** → **Create API key**
4. Скопируй ключ (выглядит как `AIzaSy...`)

> **Бесплатный лимит:** `gemini-2.0-flash-lite` — 1500 запросов/день, сбрасывается в полночь по UTC.

---

## Шаг 2 — Создать Cloudflare Worker

### 2.1 Регистрация
Зарегистрируйся на [cloudflare.com](https://cloudflare.com) (бесплатно).

### 2.2 Создать Worker
**Workers & Pages → Create → Create Worker** → дай имя → **Deploy**.

### 2.3 Вставить код
После деплоя нажми **Edit code**, замени всё содержимое:

```js
export default {
  async fetch(request, env) {
    if (request.method === 'OPTIONS') {
      return new Response(null, {
        headers: {
          'Access-Control-Allow-Origin': '*',
          'Access-Control-Allow-Methods': 'POST, GET, OPTIONS',
          'Access-Control-Allow-Headers': '*',
        },
      });
    }

    if (request.method !== 'POST') {
      return new Response('Worker is running', { status: 200 });
    }

    const text = await request.text();
    if (!text) {
      return new Response(JSON.stringify({ error: 'Empty body' }), { status: 400 });
    }

    const apiKey = env.GEMINI_API_KEY;
    const body = JSON.parse(text);
    const { model, messages, stream, max_tokens, temperature } = body;
    const geminiModel = model || 'gemini-2.0-flash-lite';

    const contents = messages
      .filter(m => m.role !== 'system')
      .map(m => ({
        role: m.role === 'assistant' ? 'model' : 'user',
        parts: [{ text: typeof m.content === 'string' ? m.content : JSON.stringify(m.content) }],
      }));

    const systemInstruction = messages.find(m => m.role === 'system');

    const geminiBody = {
      contents,
      ...(systemInstruction && {
        system_instruction: { parts: [{ text: systemInstruction.content }] },
      }),
      generationConfig: {
        temperature: temperature ?? 0.7,
        maxOutputTokens: max_tokens ?? 8192,
      },
    };

    const endpoint = stream
      ? `streamGenerateContent?alt=sse&key=${apiKey}`
      : `generateContent?key=${apiKey}`;

    const geminiUrl = `https://generativelanguage.googleapis.com/v1beta/models/${geminiModel}:${endpoint}`;

    const geminiResponse = await fetch(geminiUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(geminiBody),
    });

    if (stream) {
      return new Response(geminiResponse.body, {
        headers: {
          'Content-Type': 'text/event-stream',
          'Access-Control-Allow-Origin': '*',
        },
      });
    }

    const data = await geminiResponse.json();
    const responseText = data.candidates?.[0]?.content?.parts?.[0]?.text ?? '';

    const openAIResponse = {
      id: 'chatcmpl-gemini',
      object: 'chat.completion',
      model: geminiModel,
      choices: [{
        index: 0,
        message: { role: 'assistant', content: responseText },
        finish_reason: 'stop',
      }],
      usage: { prompt_tokens: 0, completion_tokens: 0, total_tokens: 0 },
    };

    return new Response(JSON.stringify(openAIResponse), {
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*',
      },
    });
  },
};
```

Нажми **Save and Deploy**.

### 2.4 Добавить API ключ как Secret
**Settings → Variables and Secrets → Add**:
- **Type:** Secret ⚠️ не JSON
- **Name:** `GEMINI_API_KEY`
- **Value:** твой ключ из шага 1 (без кавычек)

Нажми **Save**.

Твой Worker будет доступен по адресу:
```
https://YOUR-WORKER-NAME.YOUR-SUBDOMAIN.workers.dev
```

---

## Шаг 3 — Подключить к OpenCode

Открой или создай файл конфига:
- **Windows:** `C:\Users\ИМЯ\.config\opencode\opencode.json`
- **macOS/Linux:** `~/.config/opencode/opencode.json`

Вставь:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "gemini-proxy": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Gemini via Cloudflare",
      "options": {
        "baseURL": "https://YOUR-WORKER-NAME.YOUR-SUBDOMAIN.workers.dev/v1",
        "apiKey": "dummy"
      },
      "models": {
        "gemini-2.0-flash-lite": {
          "name": "Gemini 2.0 Flash Lite"
        },
        "gemini-2.0-flash": {
          "name": "Gemini 2.0 Flash"
        }
      }
    }
  }
}
```

Замени `YOUR-WORKER-NAME.YOUR-SUBDOMAIN` на свой URL из Cloudflare.

Перезапусти OpenCode — провайдер появится в списке моделей.

---

## Проверка работы

**Windows PowerShell:**
```powershell
$body = '{"model":"gemini-2.0-flash-lite","messages":[{"role":"user","content":"hello"}]}'
Invoke-WebRequest -Uri "https://YOUR-WORKER.workers.dev/v1/chat/completions" -Method POST -ContentType "application/json" -Body $body -UseBasicParsing | Select-Object -ExpandProperty Content
```

**Linux/macOS:**
```bash
curl -X POST https://YOUR-WORKER.workers.dev/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"gemini-2.0-flash-lite","messages":[{"role":"user","content":"hello"}]}'
```

---

## Частые проблемы

| Ошибка | Причина | Решение |
|---|---|---|
| Пустой `content` в ответе | Ключ сохранён как JSON с кавычками | Пересохранить как **Secret** |
| `429 RESOURCE_EXHAUSTED` | Исчерпан дневной лимит | Создать новый ключ или подождать до полночи UTC |
| `SyntaxError: Unexpected end of JSON` | Worker получил GET вместо POST | Запрос идёт из браузера, а не из клиента |
| Модель не появляется в OpenCode | Нет `/v1` в конце `baseURL` | Добавить `/v1` в конфиг |

---

## Поддерживаемые модели

| Модель | Лимит (бесплатно) |
|---|---|
| `gemini-2.0-flash-lite` | 1500 запросов/день |
| `gemini-2.0-flash` | 1500 запросов/день |
| `gemini-1.5-pro` | 50 запросов/день |

---

## Совместимость

Прокси работает с любым OpenAI-совместимым клиентом:
- [OpenCode](https://opencode.ai)
- [Cursor](https://cursor.sh)
- [Continue](https://continue.dev)
- Любой клиент поддерживающий кастомный `baseURL`

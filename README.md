# ai-proxy-cloudflare

Бесплатные прокси для использования Google Gemini и GitHub Copilot в [OpenCode](https://opencode.ai) и любых других OpenAI-совместимых клиентах через Cloudflare Workers.

---

## Прокси

| Прокси | Провайдер | Бесплатно | Лимит |
|---|---|---|---|
| Gemini | Google AI Studio | ✅ | 1500 запросов/день |
| GitHub Copilot | GitHub | ✅ | 200 чатов/месяц |

---

## Как это работает

```
OpenCode / любой OpenAI-клиент
        ↓
Cloudflare Worker
        ↓
Google Gemini API  /  GitHub Copilot API
```

---

# Прокси 1 — Google Gemini

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
      return new Response('Gemini Proxy is running', { status: 200 });
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

---

## Шаг 3 — Подключить к OpenCode

Открой или создай файл конфига:
- **Windows:** `C:\Users\ИМЯ\.config\opencode\opencode.json`
- **macOS/Linux:** `~/.config/opencode/opencode.json`

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "gemini-proxy": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Gemini via Cloudflare",
      "options": {
        "baseURL": "https://YOUR-WORKER.workers.dev/v1",
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

## Проверка (PowerShell)

```powershell
$body = '{"model":"gemini-2.0-flash-lite","messages":[{"role":"user","content":"hello"}]}'
Invoke-WebRequest -Uri "https://YOUR-WORKER.workers.dev/v1/chat/completions" -Method POST -ContentType "application/json" -Body $body -UseBasicParsing | Select-Object -ExpandProperty Content
```

## Частые проблемы

| Ошибка | Причина | Решение |
|---|---|---|
| Пустой `content` в ответе | Ключ сохранён как JSON с кавычками | Пересохранить как **Secret** |
| `429 RESOURCE_EXHAUSTED` | Исчерпан дневной лимит | Создать новый ключ или подождать до полночи UTC |
| `SyntaxError: Unexpected end of JSON` | Worker получил GET вместо POST | Запрос идёт из браузера |
| Модель не появляется в OpenCode | Нет `/v1` в конце `baseURL` | Добавить `/v1` в конфиг |

## Поддерживаемые модели Gemini

| Модель | Лимит (бесплатно) |
|---|---|
| `gemini-2.0-flash-lite` | 1500 запросов/день |
| `gemini-2.0-flash` | 1500 запросов/день |
| `gemini-1.5-pro` | 50 запросов/день |

---

# Прокси 2 — GitHub Copilot

## Почему нужен прокси?

OpenCode по умолчанию использует `api.githubcopilot.com` (Business/Enterprise endpoint).
Для **Individual/Free подписки** нужен `api.individual.githubcopilot.com` + специальные заголовки редактора.
Worker решает обе проблемы автоматически.

## Шаг 1 — Получить GitHub токен

1. Убедись что у тебя активна подписка [GitHub Copilot Free](https://github.com/features/copilot/plans)
2. Авторизуйся в OpenCode через `/connect` → GitHub Copilot
3. Скопируй токен из `C:\Users\ИМЯ\.local\share\opencode\auth.json` (поле `access`, начинается с `gho_`)

> ⚠️ Токен `gho_` периодически обновляется. При истечении нужно переавторизоваться и обновить секрет в Worker.

## Шаг 2 — Создать Cloudflare Worker

Создай новый Worker и вставь код:

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
      return new Response('GitHub Copilot Proxy is running', { status: 200 });
    }

    const text = await request.text();
    if (!text) {
      return new Response(JSON.stringify({ error: 'Empty body' }), { status: 400 });
    }

    const token = env.GITHUB_COPILOT_TOKEN;
    const body = JSON.parse(text);

    const copilotResponse = await fetch('https://api.individual.githubcopilot.com/chat/completions', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json',
        'Editor-Version': 'vscode/1.85.0',
        'Editor-Plugin-Version': 'copilot-chat/0.12.0',
        'Copilot-Integration-Id': 'vscode-chat',
        'User-Agent': 'GitHubCopilotChat/0.20.0',
      },
      body: JSON.stringify(body),
    });

    const data = await copilotResponse.text();

    return new Response(data, {
      status: copilotResponse.status,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*',
      },
    });
  },
};
```

Добавь секрет:
- **Type:** Secret
- **Name:** `GITHUB_COPILOT_TOKEN`
- **Value:** твой `gho_` токен

## Шаг 3 — Подключить к OpenCode

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "github-copilot": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "GitHub Copilot",
      "options": {
        "baseURL": "https://YOUR-WORKER.workers.dev/v1",
        "apiKey": "dummy"
      },
      "models": {
        "gpt-4o": {
          "name": "GitHub Copilot (GPT-4o)"
        }
      }
    }
  }
}
```

## Проверка (PowerShell)

```powershell
$body = '{"model":"gpt-4o","messages":[{"role":"user","content":"hello"}]}'
Invoke-WebRequest -Uri "https://YOUR-WORKER.workers.dev/v1/chat/completions" -Method POST -ContentType "application/json" -Body $body -UseBasicParsing | Select-Object -ExpandProperty Content
```

## Доступные модели GitHub Copilot (Free план)

| Модель | Описание |
|---|---|
| `gpt-4o` | GPT-4o от OpenAI |
| `gpt-4o-mini` | Лёгкая версия GPT-4o |
| `gpt-4.1` | GPT-4.1 от OpenAI |
| `claude-haiku-4.5` | Claude Haiku от Anthropic |
| `gpt-5-mini` | GPT-5 mini от OpenAI |

## Частые проблемы

| Ошибка | Причина | Решение |
|---|---|---|
| `forbidden: access denied` | Неправильный endpoint | Убедись что используешь Worker, а не прямой URL |
| `model_not_supported` | Модель недоступна на Free плане | Использовать модели из таблицы выше |
| `bad request: Authorization header` | Неправильный формат токена | Токен должен быть `gho_`, не `ghp_` |
| Токен перестал работать | `gho_` токен истёк | Переавторизоваться в OpenCode и обновить секрет |

---

## Совместимость

Оба прокси работают с любым OpenAI-совместимым клиентом:
- [OpenCode](https://opencode.ai)
- [Cursor](https://cursor.sh)
- [Continue](https://continue.dev)
- Любой клиент поддерживающий кастомный `baseURL`

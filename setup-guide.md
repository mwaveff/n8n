# Запуск n8n — локально і на сервері

Система: 5 workflow у цій папці + Telegram-бот. Нижче — як підняти все локально на Windows, протестувати, а потім перенести на сервер.

---

## Частина A. Встановити Docker Desktop (один раз)

1. Завантаж Docker Desktop: https://www.docker.com/products/docker-desktop/
2. Запусти інсталятор, погодься на встановлення WSL2 (запропонує саме), перезавантаж ПК якщо попросить.
3. Відкрий Docker Desktop і дочекайся статусу **Engine running** (зелений значок внизу зліва).

Перевірка: відкрий PowerShell і виконай `docker --version` — має показати версію.

---

## Частина B. Запустити n8n (одна команда)

Відкрий **PowerShell** і виконай:

```powershell
docker volume create n8n_data
docker run -d --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
```

- `n8n_data` — це сховище, де n8n тримає всі workflow і credentials (не зникне після перезапуску).
- `-d` — працює у фоні.

Відкрий у браузері: **http://localhost:5678** → створи локальний акаунт (email + пароль, лише для входу в n8n).

Зупинити / запустити пізніше:
```powershell
docker stop n8n
docker start n8n
```

---

## Частина C. Імпортувати workflow

У n8n: вгорі справа **⋮ (три крапки) → Import from File** і по черзі завантаж кожен файл із цієї папки:

- `AI News Bot — Автопостинг (AI Love You).json`
- `AI News Bot — Бот-хаб (input + модерація).json`
- `AI News Bot — Щотижневий дайджест.json`
- `AI News Bot — Error-нотифікації.json`

(`AI Agent workflow.json` — це чужий приклад, можна не імпортувати.)

---

## Частина D. Додати credentials (ключі)

Після імпорту ноди світитимуться помилкою «no credential» — це нормально, треба підставити ключі. У n8n: **Credentials → Create new**.

Потрібні:

1. **Telegram API** — токен бота від @BotFather.
   - Тип credential: *Telegram API*. Вставити токен.
   - Призначити в усіх Telegram-нодах і в Telegram Trigger.
2. **Groq для тексту** (генерація постів — безкоштовно, без картки).
   - Ключ: console.groq.com → API Keys → Create (`gsk_...`).
   - У n8n створи credential типу *OpenAI*, але в полі **Base URL** впиши `https://api.groq.com/openai/v1`, а в API Key — ключ Groq.
   - Признач у нодах «LLM генерація» (Автопостинг і Бот-хаб) та «LLM дайджест». Модель: `openai/gpt-oss-120b`.
   - Примітка: пробували OpenRouter (Claude — платний) і Gemini (через OpenAI-перехідник віддає порожнє) — не підійшли. Groq працює стабільно і безкоштовно.
3. **OpenAI для voice + обкладинок** (опційно, лише якщо потрібні ці функції).
   - OpenRouter НЕ вміє транскрипцію (Whisper) і генерацію зображень.
   - Якщо треба voice-повідомлення та авто-обкладинки — заведи окремий ключ platform.openai.com, credential типу *OpenAI* (звичайний, без Base URL), і признач у нодах «Транскрипція» та «Генерація обкладинки».
   - Альтернатива за ТЗ: Gemini для транскрипції, Flux/Leonardo для зображень — переробимо пізніше за потреби.
4. **Tavily** — вже НЕ потрібен окремо: ключ вшито прямо в ноду «Пошук Tavily».

---

## Частина E. Тест без Telegram (найшвидший)

Workflow **Автопостинг** не залежить від бота — його можна перевірити одразу:

1. Відкрий «AI News Bot — Автопостинг».
2. Клікни ноду **Пошук Tavily → Execute step** — маєш отримати JSON з `results`.
3. Клікни **Execute workflow** (внизу) — пройде весь ланцюг: джерела → фільтр → LLM → відправка на модерацію в Telegram.

Якщо пост прийшов тобі в Telegram на модерацію з кнопками — автоматична частина працює.

---

## Частина F. Увімкнути Telegram-бота локально (потрібен публічний URL)

Telegram-бот спілкується з n8n через webhook, а для нього треба адреса, доступна з інтернету. Локально це дає вбудований тунель n8n.

Перезапусти контейнер у режимі тунелю:

```powershell
docker stop n8n
docker rm n8n
docker run -d --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n start --tunnel
```

У логах (`docker logs n8n`) з'явиться публічний URL виду `https://...hooks.n8n.cloud`. n8n сам зареєструє webhook у Telegram, щойно ти **активуєш** workflow «Бот-хаб» (перемикач Active вгорі).

Після цього напиши боту в Telegram — він має відповісти/згенерувати пост.

> Тунель — лише для тестів. Для постійної роботи потрібен сервер (Частина G).

---

## Частина G. Перенесення на сервер (фінал за ТЗ)

Коли локально все працює — на сервері той самий Docker, але з постійним доменом замість тунелю.

Орієнтовний план (коли визначишся із сервером — розпишу детально під конкретний хостинг):

1. Орендувати VPS (напр. Hetzner CX22 ~4-5€/міс, Ubuntu 22.04).
2. Встановити Docker + Docker Compose.
3. Підняти n8n за `docker-compose` з:
   - доменом (напр. `n8n.твійдомен.com`) і HTTPS через Caddy/Traefik (Let's Encrypt);
   - змінними `N8N_HOST`, `WEBHOOK_URL=https://n8n.твійдомен.com/`;
   - постійним volume для даних.
4. Імпортувати ті самі workflow і credentials.
5. Активувати workflow — webhook зареєструється на постійний домен.
6. Тест 3-5 днів (пункт ТЗ).

---

## Що знадобиться під рукою

- Токен Telegram-бота (@BotFather)
- OpenAI API key (або OpenRouter, якщо переходимо на Claude/Grok)
- ID каналу `@your_channel` (перевір, що бот доданий адміном у канал)
- Tavily — вже вшито

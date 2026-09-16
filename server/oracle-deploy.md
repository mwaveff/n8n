# Безкоштовний хостинг AI News Bot-бота на Oracle Cloud (Always Free)

Мета: підняти n8n на безкоштовній віртуальній машині Oracle, яка працює 24/7 назавжди.
Стек повністю безкоштовний: **Oracle Always Free VM + DuckDNS-домен + твій `docker-compose`**.

Орієнтовний час: 30–45 хв. Технічних навичок майже не треба — усе через веб-панель + один SSH-вхід за бажанням.

---

## Крок 0. Що знадобиться під рукою

- Пошта + банківська картка (Oracle бере картку **лише для верифікації**, Always Free-ресурси не списуються).
- Telegram-бот токен (@BotFather), Groq API key, (опційно) OpenAI key, **новий** Tavily key.
- ~10 хв на реєстрацію Oracle.

---

## Крок 1. Реєстрація в Oracle Cloud

1. Відкрий https://www.oracle.com/cloud/free/ → **Start for free**.
2. Обери **регіон** (Home Region) ближче до України — напр. **Germany Central (Frankfurt)** або **UK South (London)**. Регіон потім не змінити.
3. Пройди верифікацію (пошта, телефон, картка). Дочекайся листа «your account is ready».

> Порада: якщо реєстрація «висне» на картці — спробуй іншу картку/браузер. Це частий момент.

---

## Крок 2. Створити безкоштовну VM

1. У консолі: **☰ Menu → Compute → Instances → Create Instance**.
2. **Name**: `ainewsbot-n8n`.
3. **Image**: *Edit* → **Ubuntu** → **24.04** (або 22.04).
4. **Shape** (тип машини):
   - Найкраще: **Ampere / VM.Standard.A1.Flex**, постав **1 OCPU + 6 GB RAM** (Always Free — до 4 OCPU/24 GB).
   - Якщо «out of capacity» (буває на ARM): обери **VM.Standard.E2.1.Micro** (AMD, 1 GB) — теж Always Free, для цього бота вистачить.
   - **Переконайся, що біля shape є напис «Always Free-eligible».**
5. **SSH keys**: *Generate a key pair for me* і **завантаж приватний ключ**. Можна й пропустити.
6. **Show advanced options → Management → Cloud-init / User data**:
   - Відкрий `server/oracle-cloud-init.yaml`, заміни `AI News Bot.duckdns.org` на свій майбутній DuckDNS-субдомен.
   - Встав увесь текст файлу в це поле.
7. **Create**. За 1–2 хв машина стане *Running*. Скопіюй її **Public IP address**.

Cloud-init сам поставить Docker, відкриє порти 80/443 в ОС і підніме n8n + Caddy.

---

## Крок 3. Відкрити порти в мережі Oracle (обов'язково!)

Cloud-init відкрив порти в ОС, але Oracle має ще й **мережевий фаєрвол** (Security List):

1. На сторінці інстансу → блок **Primary VNIC** → клікни назву **Subnet**.
2. **Security Lists → Default Security List → Add Ingress Rules**.
3. Додай два правила:
   - Source CIDR `0.0.0.0/0`, TCP, порт **80**
   - Source CIDR `0.0.0.0/0`, TCP, порт **443**
4. Save.

Без цього кроку сайт не відкриється й HTTPS не видасться.

---

## Крок 4. Безкоштовний домен через DuckDNS

1. https://www.duckdns.org → залогінься (Google/GitHub).
2. Придумай субдомен, напр. `ainewsbot` → **add domain** → отримаєш `your-domain.duckdns.org`.
3. У полі **current ip** впиши **Public IP** своєї VM → **update ip**.
4. Субдомен має збігатися з `DOMAIN` у cloud-init (крок 2.6).

> Caddy автоматично випустить безкоштовний HTTPS (Let's Encrypt), щойно домен вкаже на IP (1–2 хв).

---

## Крок 5. Перший вхід у n8n

1. Відкрий `https://your-domain.duckdns.org` (свій домен).
2. n8n попросить створити **обліковий запис власника** (email + пароль) — захищає редактор. Збережи дані.

---

## Крок 6. Імпорт workflow і ключі

**⋮ → Import from File** — 4 файли: `ainewsbot-bot-hub.json`, `ainewsbot-autoposting.json`, `ainewsbot-weekly-digest.json`, `ainewsbot-error-notifications.json`.

**Credentials → Create new** (детальніше — `setup-guide.md`):

1. **Telegram API** — новий токен @BotFather. Признач у всіх Telegram-нодах і Trigger.
2. **Groq** — тип *OpenAI*, Base URL `https://api.groq.com/openai/v1`, ключ `gsk_...`. Модель `openai/gpt-oss-120b`.
3. **Tavily** — тип **Header Auth**: Name=`Authorization`, Value=`Bearer <новий_ключ>`, назва **Tavily**.
4. **OpenAI** (опційно) — для транскрипції voice та обкладинок.

---

## Крок 7. Запуск і перевірка

1. Додай бота **адміністратором** у канал `@your_channel`.
2. Увімкни **Active** у кожному workflow.
3. Тест: «AI News Bot — Автопостинг» → **Execute workflow** → пост на модерацію → «Опублікувати» → канал.
4. Напиши боту в Telegram — має відповісти (перевірка вебхука).

---

## Важливі нотатки

- **Ротація ключів**: використай нові ключі — старі були в git-історії.
- **Ключ шифрування n8n**: створюється у волюмі `n8n_data` при першому запуску. Для відновлення credentials збережи `/home/node/.n8n/config` у безпечне місце (НЕ в git).
- **IP**: публічний IP Oracle зазвичай не змінюється; якщо зміниться — онови на duckdns.org.
- **Часовий пояс**: у compose `Europe/Kyiv`. Для Варшави — заміни на `Europe/Warsaw` і `docker compose up -d`.
- **Оновлення**: `cd /opt/n8n && docker compose pull && docker compose up -d`.
- **Логи**: `docker compose logs -f n8n`.

---

## Якщо сайт не відкривається

1. Порти 80/443 в **Security List** (крок 3)?
2. DuckDNS вказує на правильний Public IP?
3. `DOMAIN` у `/opt/n8n/.env` = субдомен DuckDNS?
4. `docker compose ps` — обидва *Up*?
5. Дай Caddy 1–2 хв; `docker compose logs caddy`.

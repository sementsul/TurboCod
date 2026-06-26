# deepseek-claude — заметки проекта

Одностраничный браузерный AI-клиент (OpenAI-совместимый) + опциональный бэкенд-прокси.

## Состав
- `index.html` — самодостаточный фронт. Возможности:
  - Чат со стримингом, тёмная/светлая тема, выбор провайдера.
  - **Профили ИИ** (левая панель): несколько подключений (провайдер/URL/модель/ключ), переключение.
    Тема и системный промпт — общие. Кнопка «💾 Сохранить настройки».
  - **Мультичаты**: создание/удаление/переключение; история в localStorage (`ds_chats`),
    указатель активного чата — в куках+localStorage. (Куки ~4 КБ — всю историю не вмещают, поэтому
    она в localStorage; в куках только указатели.)
  - **Загрузка файлов** в чат (📎): текстовые инлайнятся в сообщение; картинки → data URL (vision).
  - **Наборы промптов**: встроенные (35) + файлы из `prompts/` (манифест `prompts/index.json`).
    Два выпадающих списка: набор → пресет. На file:// — только встроенные.
  - **Artifacts**: пресет «Artifacts (генератор кода)» → ИИ отвечает JSON `{response, artifacts[], suggestions[]}`.
    Панель справа: вкладки файлов, живое превью HTML/SVG (sandbox iframe), редактор, экспорт в ZIP
    (свой ZIP-упаковщик на чистом JS, без зависимостей).
  - **Функции (tool-calls)**: тумблер «🔧 Функции» — клиент сам вызывает CORS-дружелюбные API
    (get_weather через open-meteo, get_time, fetch_url). Без стриминга, нужна модель с tool-calls.
    Произвольные URL из браузера ограничены CORS — общий случай решает бэкенд `/tools/chat`.
  - Хранилище: куки + localStorage. На `file://` ненадёжно — запускать через http-сервер / GitHub Pages.
- `prompts/` — наборы промптов: `index.json` (манифест), `claude-code-models.json` (выбор «модели Claude Code»),
  `roles.json` (роли разработки). `prompts.json` в корне — legacy-набор (тоже подхватывается).
- `backend/` — FastAPI-прокси (см. backend/README.md).

## Бэкенд (backend/)
- `POST /v1/chat/completions` — OpenAI-совместимый прокси со стримингом. Прячет ключ LLM.
- `POST /tools/chat` — Function Calling: инструменты get_weather (open-meteo), fetch_data (SSRF-защита),
  search_web (DuckDuckGo IA). Бэкенд сам гоняет цикл tool_calls.
- `GET /health`.
- Безопасность: токен-гейт CLIENT_TOKENS, CORS по ALLOWED_ORIGINS, rate-limit на IP, SSRF-guard.
  Реальный ключ LLM — только в .env (в .gitignore). **Секреты — красная зона, под ревью ПМ.**
- Тесты: `backend/test_app.py` (чистая логика, без сети) — 5/5 зелёные.

## Архитектура для GitHub Pages
GitHub Pages = только статика. Фронт хостится на github.io, бэкенд — отдельно (Render/Railway/Fly).
Фронт ставит Base URL = `https://<бэкенд>/v1`, в «API-ключ» — клиентский токен (CLIENT_TOKENS).

## Как запускать локально
- Фронт: `python3 -m http.server 8000 --bind 0.0.0.0` → http://localhost:8000
- Бэк: `cd backend && uvicorn app:app --host 0.0.0.0 --port <свободный>` (нужен .env с LLM_API_KEY)

## Грабли
- На `file://` куки не сохраняются, localStorage часто тоже → настройки слетают. Решение: http-сервер/Pages.
- OpenAI блокирует CORS из браузера; DeepSeek/OpenRouter/Groq — нет. Через бэкенд-прокси CORS не проблема.
- Публичный прокси БЕЗ токен-гейта = любой сольёт твой платный баланс. CLIENT_TOKENS обязателен в проде.

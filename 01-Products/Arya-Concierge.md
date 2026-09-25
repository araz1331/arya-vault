# Arya Concierge
## Домен
concierge.arya.az → `2.28.100.162`
## Статус
🟢 В проде — 25.09.2026 переехал с Replit на [[Hetzner-Server]].
Деплой на Replit снят с публикации.
## Архитектура
- Repo: `araz1331/concierge`
- `/var/www/arya-concierge`, PM2 (`arya-concierge`), порт 3030, nginx + HTTPS
  (Let's Encrypt, до 24.12.2026)
- Node 24 из `/opt/node24` — на машине системный Node 20, на котором работают
  [[Arya-Mail]] и `kapital-proxy`, и трогать его не стали
- Монорепозиторий pnpm: Express 5 + React/Vite + Drizzle
- **База — общая с [[Arya-Hub]]:** `wzkdfqpfzndywxqvqaqn`. Прежний проект
  `jknvrkevqqoaoiemhznt` цел, но больше не читается.
## Позиционирование
B2C-агент для Азербайджана в духе Instinct: не чат-бот, а **консьерж-сервис**.
Формулировка не косметическая — см. [[Agent-Memory-Research]]: Meta с 15.01.2026
запрещает в WhatsApp универсальных AI-ассистентов, и Азербайджан в исключения
не входит.
## Агенты
✅ Press, Broker, Legal, Tax (Muhasibat)
🔜 Executive, Uni
## Открытые задачи
- [ ] Слияние с [[Arya-Hub]] в один продукт — крупная задача, следующая сессия
- [ ] Таблицы памяти в общей базе: `user_memory`, `user_episodes`,
      `user_playbooks`, `autonomy_settings` — см. [[Agent-Memory-Research]]
- [ ] Активные действия: бронирование, звонки в заведения
- [ ] Заполнить `TRAVELPAYOUTS_MARKER` — сейчас пусто, ссылки на авиабилеты
      уходят без партнёрской метки
## Связанное
[[Arya-Hub]] · [[Hetzner-Server]] · [[Agent-Memory-Research]] · [[Arya-Connect]]

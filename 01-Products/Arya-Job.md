# Arya Job (Arya İş)
## Домен
job.arya.az
## Статус
🟡 Миграция на Cloudflare (DNS переключается)
## Архитектура (новая, с 19.09.2026)
- Repo: araz1331/job | Ветка: `migrate/cloudflare-stack` (в main НЕ смержена)
- Стек: Hono Workers + Cloudflare Pages + Drizzle/Supabase HTTP
- Монорепо: apps/web (Pages, статика) | workers/api (Hono) | packages/db | scripts (скрапер)
- Worker: `arya-job-api-production` ✅ задеплоен 19.09 19:48 UTC, все 8 секретов стоят
- Route `job.arya.az/api/*` прописан в wrangler.toml, но ещё НЕ активен (DNS на Replit)
- Скрапер: переносится в GitHub Actions cron, `.github/workflows/scrape.yml` 22:00 UTC / 02:00 Baku
- Supabase: `xrozowxzcofdoemsmfzk` (eu-central-1) → [[Supabase-Org]]
- Агент arya-job за [[Arya-Concierge]], reply → `/api/agent-reply` (не `/api/reply`)
- Подробности миграции: [[Migration-Cloudflare]]
## ⚠️ Проверено 20.09
- job.arya.az → 34.111.179.208 (Replit/GCP), отдаёт «This app isn't live yet» — **сайт сейчас лежит**
- `/api/health` недоступен: трафик не доходит до Cloudflare, пока DNS не переключён
## Старый стек (Replit, до 19.09)
- Express + node-cron + postgres-js TCP | код сохранён в `server/` как референс
- ⚠️ Перед Republish: git pull origin main в Replit Shell
## Открытые задачи
- [ ] DNS job.arya.az → Cloudflare (Pages + route на Worker)
- [ ] DB соединение: env-check утром — проверить drizzle_query() и rate_limits в Supabase
- [ ] Применить `packages/db/sql/0000_drizzle_query.sql` и `0001_rate_limits.sql`
- [ ] Смержить ветку в main — GitHub берёт `schedule:` только из default-ветки, иначе cron скрапера не запустится
- [ ] Создать Pages-проект (Git-интеграция только через дашборд, repo `araz1331/job`)
## Решения
Телефоны работодателей — любой международный формат, не только +994
Рейт-лимиты OTP переехали из памяти процесса в таблицу `rate_limits` (у каждого изолята Workers своя память)

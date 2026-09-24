# Миграция на Cloudflare
## Цель
Уйти с Replit полностью, экономия ~$200/мес
## Новый стек
Cloudflare Pages (статика) + Workers (API, Hono) + Supabase HTTP + Drizzle
Шаблон: github.com/araz1331/arya-template
## Ключевое ограничение
У Workers нет TCP-сокетов → `postgres-js`/`pg` там не работают.
Drizzle остаётся, но ездит по HTTPS: `pg-proxy` → supabase-js `.rpc()` → SQL-функция `drizzle_query()`.
Функция выполняет произвольный SQL, поэтому выдана **только service_role** → RLS на этом пути обходится, авторизация целиком на Worker.
Долгие задачи (скраперы) в Worker не влезают → GitHub Actions, там TCP есть.
## Статус сервисов
- 🟢 hr.arya.az — [[Arya-HR]] (Pages, hr-3jd.pages.dev, проверено 20.09)
- 🟢 connect.arya.az — [[Arya-Connect]] (Workers + Pages, собран 22–24.09)
- 🟡 job.arya.az — [[Arya-Job]] (Worker задеплоен + секреты, DNS ещё на Replit, сайт лежит)
- 🔴 concierge.arya.az — [[Arya-Concierge]]
- 🔴 tax.arya.az (Arya Muhasibat)
- 🔴 legal.arya.az — [[Arya-Legal]]
- 🔴 press.arya.az — [[Arya-Press]]
- 🟢 wa.arya.az — [[Arya-Hub]] (ушёл с Replit 22.09, но **не** на Cloudflare:
  Docker на [[Hetzner-Server]] — шлюзу нужны долгоживущие процессы)
- 🟢 mail.hirearya.com — [[Arya-Mail]] (там же, PM2)
- 🔴 broker — [[Arya-Broker]]
## Чеклист на сервис
- [ ] Монорепо по шаблону: apps/web + workers/api + packages/db
- [ ] Express-роуты → Hono, ответы и статус-коды оставить байт-в-байт (Concierge на них завязан)
- [ ] `postgres-js` → Drizzle/Supabase HTTP; применить `0000_drizzle_query.sql`
- [ ] Состояние из памяти процесса → в таблицы (рейт-лимиты, кэши): у каждого изолята своя память
- [ ] `process.env` → типизированные bindings, читать только внутри запроса
- [ ] Крон из node-cron → GitHub Actions (или Cron Trigger, если влезает в лимиты)
- [ ] Секреты: `wrangler secret put --env production`
- [ ] DNS → Cloudflare, проверить что route Worker'а активен
## Грабли (собраны на job)
- Pages-проект с Git-интеграцией создаётся **только через дашборд**, `wrangler pages project create` делает Direct Upload без авто-деплоя
- Worker route не стреляет, пока DNS не на Cloudflare — трафик просто не доходит
- `schedule:` в GitHub Actions запускается только из default-ветки → в feature-ветке cron молчит
- `wrangler secret put` на несуществующий Worker уходит в интерактивный промпт → сначала deploy
- Pages `_redirects` не умеет проксировать на чужой origin → на `*.pages.dev` превью API недоступен
- pnpm 11: `onlyBuiltDependencies` переименован в `allowBuilds`, иначе install падает и портит pnpm-workspace.yaml
- Один `catalog:` на версию в pnpm-workspace.yaml — две копии drizzle-orm дают тысячи фейковых ошибок типов
## Куда Cloudflare не подходит
Медиасервер голоса, SMTP и всё, что живёт дольше запроса, уехало на
[[Hetzner-Server]], а не в Workers. Цель «уйти с Replit» от этого не страдает —
страдает только формулировка «всё на Cloudflare».

## Связанное
[[Supabase-Org]] · [[Hetzner-Server]]

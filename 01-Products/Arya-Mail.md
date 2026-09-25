# Arya Mail
## Домен
mail.hirearya.com
## Статус
🟢 В проде — 24.09.2026 переехал с Replit на [[Hetzner-Server]]
## Архитектура
- Repo: araz1331/Arya-Mail
- Express/TypeScript, `/var/www/arya-mail`, PM2 + `ecosystem.config.cjs`
- Supabase: `pypvjnzlkmoikfzhuwbm`
- Провайдер: **Resend** (Pro, $20/мес, аккаунт dagik). SES пробовали и откатили.
## Исходящие
`POST /api/v1/notify`, отправка с `noreply@hirearya.com`.
## Входящие
MX `mail.hirearya.com` → `inbound-smtp.eu-west-1.amazonaws.com` (Resend работает
на инфраструктуре SES — это нормально, не путать с SES-аккаунтом).
Вебхук `email.received` несёт **только метаданные**, тело забирается отдельно по
`email_id`. Адаптер: `POST /api/v1/inbound/resend`, подпись Svix.
Адресация по слагу арендатора: `<slug>@mail.hirearya.com` → [[Arya-Connect]].
## Домены
`hirearya.com` и `mail.hirearya.com` — верифицированы (SPF/DKIM/TXT).
TLS от Let's Encrypt, истекает 23.12.2026 (проверено 25.09).
## Грабли
- В приложении **нет dotenv**: `.env` сам по себе не читается, окружение
  прокидывает `ecosystem.config.cjs`. PM2 при этом показывает `online`, хотя
  порт никто не слушает.
- Первая версия маршрутизации входящих рассылала письмо во **все** зарегистрированные
  вебхуки (6 подписчиков) — утечка между арендаторами. Закрыто
  `TENANT_ROUTER_WEBHOOK_URL`, падает закрыто.
- Проверка SPF должна разбирать `include:`, а не искать подстроку; DKIM у Resend —
  селекторы `resend`/`send`, запись без `v=DKIM1`, но с непустым `p=`.
## Открытые задачи
- [ ] Ротировать засвеченные ключи Resend
- [ ] Кнопка «проверить домен» в панели [[Arya-Connect]]

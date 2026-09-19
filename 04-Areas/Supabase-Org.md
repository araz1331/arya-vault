# Supabase — организации
## ARYA AI BOS (Pro)
Org ID: `vxbqgslosdqjpibntpua`
Все продуктовые проекты живут здесь.
## Проекты
- Arya HR — `xxzyutniiqmyrihhpooh` → [[Arya-HR]]
- Arya Job — `xrozowxzcofdoemsmfzk` (eu-central-1) → [[Arya-Job]]
- Arya Press → [[Arya-Press]]
- broker → [[Arya-Broker]]
- Concierge → [[Arya-Concierge]]
- wa.arya.az (WhatsApp Hub)
- Arya Chat Voice Sales
- arya-legal → [[Arya-Legal]]
(ref'ы проставлены только для HR и Job — остальные добавить при миграции)
## id4.art
Отдельная **Free** org, не в ARYA AI BOS → [[ID4-ART]]
## Доступ из Cloudflare Workers
Drizzle ходит по HTTPS через `drizzle_query()`, выдана только service_role — ключ service_role
живёт в секретах Worker'а и **никогда** не уезжает в браузер. Подробности: [[Migration-Cloudflare]]
## Открытые задачи
- [ ] Проставить project ref'ы для остальных проектов
- [ ] Проверить, где ещё используется прямое TCP-подключение (миграции, скраперы)

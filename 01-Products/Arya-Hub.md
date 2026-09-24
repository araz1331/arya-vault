# Arya Hub
## Домен
wa.arya.az
## Статус
🟢 Центральный шлюз экосистемы · с 22.09.2026 на [[Hetzner-Server]] (Docker), Replit отключён
## Архитектура
- Repo: araz1331/WhatsApp-Gateway
- Supabase: wzkdfqpfzndywxqvqaqn
- Node.js/Express/TypeScript, Gemini Flash classifier
- 3 Meta WABA номера
- Батчинг: 15с тишины для текста, мгновенно для медиа
- Контейнер `arya-hub-api`; логи — `docker logs arya-hub-api`
## Чистка 23.09
Выпилены Twilio (−1358 строк), Swarovski, Green API — шлюз остался только на Meta.
## Инцидент 23.09
JWT `service_role` утёк в репозиторий → ключ ротирован, история git переписана.
## Meta BSP
App Review подан 07.08 — первый BSP в Азербайджане после одобрения
## Подключённые продукты
concierge, Legal, Muhasibat, Uni, Chat, Executive, Real Estate AZ, Press, [[Arya-Connect]]
## Шаблоны на общем WABA (1410955187551881)
Общий номер, с которого шлют арендаторы без своего WABA.
- `arya_connect_otp` (az), `arya_connect_otp_ru` (ru) — вход в [[Arya-Connect]]
- `arya_notification` — алерт менеджеру, когда AI просит помощи
- `arya_manager_invite` — приглашение менеджера
## Открытые задачи
- [ ] Подтянуть на сервере фикс юникода в SMS (коммит 4038a38) — пересобрать контейнер
- [ ] Транспорт WhatsApp bulk (BULK_SMS2)

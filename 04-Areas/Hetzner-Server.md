# Hetzner — сервер
## Машина
CPX42, Falkenstein, Ubuntu 24.04, `2.28.100.162` (создан 22.09.2026)
Заведён под то, что не влезает в Cloudflare Workers: долгоживущие процессы,
медиасервер, SMTP. См. ограничения в [[Migration-Cloudflare]].
## Что крутится
- [[Arya-Hub]] — `wa.arya.az`, Docker, контейнер `arya-hub-api`
- [[Arya-Mail]] — `/var/www/arya-mail`, PM2, `mail.hirearya.com`
- [[Arya-Concierge]] — `/var/www/arya-concierge`, PM2, порт 3030,
  `concierge.arya.az`, HTTPS до 24.12.2026. Node 24 из `/opt/node24`:
  системный Node 20 остался под arya-mail и kapital-proxy.
- `arya-voice-agent` — Docker, голосовой агент WhatsApp
- LiveKit — медиасервер (контейнер `livekit-livekit-1`). Отдельного coturn на
  машине **нет** — вопреки записи от 22.09: `turnserver` не установлен, юнит
  неактивен, TURN обслуживает сам LiveKit. Проверено 25.09.
- `kapital-proxy` — PM2, 127.0.0.1:3020, `kapital.hirearya.com`. Платёжные
  вызовы Connect идут отсюда, потому что у машины статический IP, который банк
  может внести в договор. См. [[Kapital-Payments]]
## Обслуживание
- HTTPS через nginx + certbot
- Логи Hub: `docker logs arya-hub-api`
- Логи Mail: `pm2 logs arya-mail`
## Грабли
- PM2 `restart --update-env` **не** перечитывает ecosystem-файл; новые ключи
  подхватывает только `pm2 start ecosystem.config.cjs`
- `pm2 env <id>` показывает не то окружение, что у процесса — смотреть
  `/proc/<pid>/environ`
- systemd-resolved кэширует отрицательные ответы DNS → проверку доменов делать
  через публичные резолверы
- Прямой хост Supabase (`db.<ref>.supabase.co`) отвечает **только по IPv6** —
  с ноутбука не резолвится, дампы и psql гонять отсюда
- `new Pool({ connectionString })` в node-postgres включает TLS **только** если
  в строке есть `sslmode`. Строка Hub его не содержала: без правки Concierge
  ходил бы в базу открытым текстом. Проверено — `ssl=false` против `TLSv1.3`.
## Открытые задачи
- [ ] Пересобрать `arya-hub-api` — фикс юникода в SMS (4038a38) подтянут в
      рабочую копию 25.09, но образ не пересобран, значит не в проде
- [ ] DNS `kapital.hirearya.com` → 2.28.100.162, затем certbot
- [ ] Бэкапы и мониторинг машины

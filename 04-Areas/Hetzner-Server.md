# Hetzner — сервер
## Машина
CPX42, Falkenstein, Ubuntu 24.04, `2.28.100.162` (создан 22.09.2026)
Заведён под то, что не влезает в Cloudflare Workers: долгоживущие процессы,
медиасервер, SMTP. См. ограничения в [[Migration-Cloudflare]].
## Что крутится
- [[Arya-Hub]] — `wa.arya.az`, Docker, контейнер `arya-hub-api`
- [[Arya-Mail]] — `/var/www/arya-mail`, PM2, `mail.hirearya.com`
- LiveKit + coturn — медиасервер для WhatsApp Voice
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
## Открытые задачи
- [ ] Пересобрать Hub после фикса юникода в SMS (4038a38) и бэклога voice-agent (f9ac80e)
- [ ] Бэкапы и мониторинг машины

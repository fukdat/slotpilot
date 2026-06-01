# SlotPilot

Движок бронирования и расписаний для сервисных бизнесов (салоны, клиники, репетиторы,
студии) с **защитой от двойного бронирования из коробки**. Клиенты видят доступность в
реальном времени; в календарь сотрудника никогда не попадают две записи в один слот.

> Заметка для портфолио: чистая архитектура **ports & adapters**, чистый
> детерминированный движок расписаний, строгий TypeScript и полное покрытие
> unit- и e2e-тестами.

## Стек

- **TypeScript** (strict, `noUncheckedIndexedAccess`) · **Node 22**
- **NestJS** (HTTP) · валидация запросов **class-validator**
- **Jest** + **supertest** (unit + e2e) · **Docker** · **GitHub Actions**

## Архитектура

```
src/
  domain/          чистая логика расписаний — без фреймворка, без I/O
    time.ts        полуоткрытые интервалы, пересечения, UTC-хелперы
    scheduling.ts  генерация слотов + правила бронируемости
    booking.ts     сущности (Service, Resource, Booking)
  application/     use-cases + порты (интерфейсы репозитория/часов)
  infrastructure/  адаптеры (in-memory репозитории; в проде меняются на Postgres)
  http/            NestJS-контроллер, DTO, фильтр доменная-ошибка → HTTP
```

**Движок расписаний чистый**: одни и те же входные данные всегда дают одни и те же
слоты, поэтому он тривиально тестируется. **Инвариант отсутствия пересечений**
обеспечивается атомарно на границе репозитория — в продакшене это отображается в
Postgres-ограничение исключения
`EXCLUDE USING gist (resource_id WITH =, tstzrange(start,end) WITH &&)`.

### Гарантии корректности (покрыты тестами)

- Полуоткрытые интервалы: смежные брони (09:00–09:30, 09:30–10:00) не сталкиваются,
  но любое реальное пересечение отклоняется.
- Слоты в прошлом, вне рабочих часов или пересекающие UTC-полночь отклоняются.
- Забронированные слоты исчезают из доступности; отмена снова их освобождает.
- Одновременные идентичные запросы не могут оба пройти (единый check-and-set).

## Запуск

```bash
npm install
npm run start:dev      # http://localhost:3000
```

### Попробовать

```bash
# Demo-данные: ресурс "res_alex", услуги "svc_haircut" (30м) / "svc_color" (90м),
# доступны Пн–Пт 09:00–17:00 UTC.
curl "localhost:3000/availability?resourceId=res_alex&serviceId=svc_haircut&day=2026-06-15"

curl -X POST localhost:3000/bookings -H 'content-type: application/json' -d '{
  "resourceId":"res_alex","serviceId":"svc_haircut",
  "startsAt":"2026-06-15T09:00:00Z",
  "customerName":"Jane","customerEmail":"jane@example.com"
}'
```

### Docker

```bash
docker build -t slotpilot . && docker run -p 3000:3000 slotpilot
```

## Гейты качества

```bash
npm run typecheck   # tsc --strict, без эмита
npm test            # jest unit + e2e
npm run build       # компиляция в dist/
```

## Фронтенд (`web/`)

Публичная страница бронирования на **Next.js 14** (App Router, TypeScript, Tailwind) с
тёплым редакционным дизайном — четырёхшаговый поток (услуга → день → время → детали),
который читает доступность вживую и мгновенно подтверждает запись через это API.

```bash
# 1. запустить API (CORS включён для origin фронтенда)
npm run build && PORT=3000 npm start

# 2. запустить веб-приложение
cd web && npm install
NEXT_PUBLIC_API_URL=http://localhost:3000 npm run dev   # http://localhost:3001
```

Задайте `CORS_ORIGINS` на API (через запятую), чтобы ограничить разрешённые origin в
продакшене; по умолчанию `*`.

## Дорожная карта

- [ ] Адаптер TypeORM + Postgres с ограничением исключения `tstzrange`
- [ ] Обработка таймзоны организации на границе API (хранить UTC)
- [ ] Уведомления (email / SMS / Telegram) при бронировании и напоминания
- [ ] Онлайн-оплата (депозит) при подтверждении брони
- [x] Публичная страница бронирования на Next.js (`web/`)

## Лицензия

MIT

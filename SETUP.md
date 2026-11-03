# Подключение сервера клуба (Supabase)

Без этих настроек работает всё, кроме клуба и отправки отзывов на сервер.

## 1. Создайте проект

1. Создайте проект на https://supabase.com.
2. В разделе Authentication → Providers (Sign In / Providers) включите **Anonymous sign-ins**.

## 2. Создайте таблицы

Выполните в SQL Editor:

```sql
create table public.players (
  id uuid primary key default auth.uid() references auth.users(id) on delete cascade,
  data jsonb not null,
  updated_at timestamptz not null default now()
);

create table public.feedback (
  id bigint generated always as identity primary key,
  data jsonb not null,
  created_at timestamptz not null default now()
);

alter table public.players enable row level security;
alter table public.feedback enable row level security;

-- Участники клуба видят записи друг друга, но меняют только свою
create policy "players read" on public.players for select to authenticated using (true);
create policy "players insert" on public.players for insert to authenticated with check (id = auth.uid());
create policy "players update" on public.players for update to authenticated using (id = auth.uid()) with check (id = auth.uid());
create policy "players delete" on public.players for delete to authenticated using (id = auth.uid());

-- Отзывы можно только отправить; читает владелец в панели Supabase
create policy "feedback insert" on public.feedback for insert to authenticated with check (true);

alter publication supabase_realtime add table public.players;
```

## 3. Впишите ключи

В `config.js` заполните `supabaseUrl` и `supabaseKey` (ключ **anon public**). Этот ключ рассчитан на публикацию, защиту обеспечивает RLS. Ключ `service_role` в репозиторий добавлять нельзя.

## Что проверить до запуска

Эта схема и подключение ещё не проверялись на живом проекте.

- **Экранирование данных.** Часть полей других участников (например, `ava`, `stake`, `key`) выводится в HTML без экранирования. Пока сервер не подключён, это неопасно. С общим сервером участник сможет подставить в свою запись вредоносный код, поэтому до запуска такие поля нужно экранировать или проверять.
- **Локация.** Таблица `players` доступна на чтение всем участникам, поэтому локация видна при прямом запросе к базе (см. раздел 9 PRD).
- **Отзывы и контакты.** Админ-панель в приложении работает только в онлайн-версии. С Supabase записи читаются в панели Supabase (таблица `feedback`).
- **Юридические документы.** Перед публичным запуском нужны документы из раздела 8 PRD.

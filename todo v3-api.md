# v3-api: Полный список нереализованного

---

## 1. Категории: дерево + классификация сущностей

**Проблема:** Сейчас все сущности наследуют категорию заметки. PostgreSQL получает «Здоровье».

**Что нужно:**
1.1. `parent_id` в таблицу categories (ALTER TABLE + миграция)
1.2. Заполнить подкатегории: Здоровье→сон, Здоровье→спина, Здоровье→зубы, Работа→технологии, Работа→проекты, Финансы→доходы, Финансы→расходы…
1.3. Написать `classify_entity(name, entity_type)` — keyword match по имени, entity_type как подсказка
1.4. В pipeline: заменить `assign_entity_category(entity_id, note_category)` на вызов `classify_entity()`. Каждая сущность получает свои категории
1.5. Обновить `compute_priorities()`: приоритет учитывает вес подкатегории + родителя

**Файлы:** db.py (schema, функция), graph_builder.py или main.py (pipeline), priority.py (формула)

---

## 2. relation_stats — пустая таблица

**Проблема:** Таблица `relation_stats` создана, но никогда не заполняется.

**Что нужно:**
2.1. Написать `update_relation_stats()` — агрегация по парам (from, to): сколько раз встретилась, в скольких notes, first_seen, last_seen, confidence
2.2. Интегрировать в pipeline (после sync_graph_to_pg)

**Файлы:** db.py (функция + schema), main.py (pipeline)

---

## 3. Observations — дубликаты при повторной обработке

**Проблема:** `insert_note` возвращает существующий note_id (по хэшу), но `insert_observation` каждый раз добавляет новые строки. При повторной отправке той же заметки observations удваиваются.

**Что нужно:**
3.1. `delete_observations_by_note(note_id)` — очистка старых observations для note_id
3.2. Вызывать перед `insert_observation` в process и pipeline

**Файлы:** db.py, main.py

---

## 4. Пагинация

**Проблема:** `get_all_entities()`, `get_all_relations()`, `get_all_actions()` загружают всё в память. При 10k+ записей упадёт.

**Что нужно:**
4.1. Добавить параметры `limit=100, offset=0` во все get_all_* функции
4.2. Обновить endpoint'ы: `/v3/entities?limit=50&offset=0`

**Файлы:** db.py, main.py

---

## 5. Legacy relations без note_id

**Проблема:** 195 relations созданы ДО миграции и не имеют note_id. Их вес не дедуплицируется — каждая строка даёт +1.

**Что нужно:**
5.1. Скрипт backfill: попытаться привязать legacy relations к существующим notes (по совпадению текста)
5.2. Или просто удалить legacy relations, если notes переобработаны через pipeline

**Файлы:** отдельный скрипт миграции

---

## 6. update_entity_stats — неточный LIKE match

**Проблема:** `LOWER(o.text) LIKE '%' || LOWER(e.name) || '%'` может давать ложные совпадения (например, «сон» найдётся в «персона»).

**Что нужно:**
6.1. Заменить на tsvector/tsquery (полнотекстовый поиск PostgreSQL)
6.2. Или добавить проверку по границам слов (regexp)

**Файлы:** db.py

---

## 7. n8n — полный end-to-end тест

**Проблема:** webhook отвечает, но не проверено, что workflow доходит до v3-api и обрабатывает ответ.

**Что нужно:**
7.1. Проверить импорт workflow `v3-full-stack.json`
7.2. Проверить доступность v3-api из контейнера n8n (docker network)
7.3. Протестировать цепочку: webhook → HTTP node → v3-api → response → LightRAG → PG

**Файлы:** n8n_workflows/*.json, docker-compose.yml (networks)

---

## 8. Приоритет — ручные действия без FK

**Проблема:** `compute_priorities()` делает `continue` для действий без `action_entities`. Ручные действия не получают приоритет.

**Что нужно:**
8.1. Для ручных действий (нет action_entities) — fallback: поиск по описанию (word matching)
8.2. Или при создании ручного действия — автоматически линковать к сущностям

**Файлы:** priority.py

---

## 9. Кэш: top_actions и рекомендации

**Проблема:** Уровень 3 (Cache) по документу должен содержать готовые результаты, а не вычислять их каждый раз.

**Что нужно:**
9.1. Таблица `cached_top_actions` — хранить топ-10 с timestamp
9.2. Инвалидировать при новом pipeline / compute
9.3. Endpoint `/v3/cache/top-actions` — отдаёт из кэша

**Файлы:** db.py, main.py

---

## 10. Тесты (интеграционные)

**Проблема:** Нет автоматического запуска тестов при сборке.

**Что нужно:**
10.1. Добавить `test_runner.py` в сборку
10.2. `docker compose run v3-api python3 /app/app/test_runner.py`
10.3. CI-ready

**Файлы:** Dockerfile, test_runner.py

---

## Приоритеты

| # | Что | Важность | Время |
|---|-----|----------|-------|
| 1 | Категории: дерево + classify_entity | 🔴 Критично | ~2ч |
| 2 | relation_stats | 🔴 Критично | ~30м |
| 3 | Observations dedup | 🟡 Важно | ~15м |
| 4 | Пагинация | 🟡 Важно | ~20м |
| 5 | Legacy relations | ⚪ Нормально | ~1ч |
| 6 | LIKE → tsvector | ⚪ Нормально | ~30м |
| 7 | n8n end-to-end | ⚪ Нормально | ~1ч |
| 8 | Ручные actions | 🔵 По желанию | ~30м |
| 9 | Кэш | 🔵 По желанию | ~30м |
| 10 | CI-тесты | 🔵 По желанию | ~30м |

# CLAUDE.md — Инструкции для Claude в проекте health_food

## Контекст проекта

Персональная система планирования питания для двух человек:
- **Сергей** (183 см) — строгий медицинский рацион от диетолога, значительное снижение веса
- **Эльвира** (173 см) — рекомендации фитнес-тренера, умеренное снижение веса

Они живут вместе и едят вместе — меню совмещённое, но калорийность и ограничения разные.

## Файлы с контекстом — всегда читать перед генерацией меню

| Файл | Что содержит |
|---|---|
| `profiles/sergey.json` | Текущий вес Сергея — **обновляется пользователем** |
| `profiles/elvira.json` | Текущий вес Эльвиры — **обновляется пользователем** |
| `rules/sergey_diet.md` | Медицинские ограничения + разрешённые продукты |
| `rules/elvira_diet.md` | Нормы тренера (1700 ккал, БЖУ) |
| `equipment/equipment.md` | Доступная техника и методы готовки |
| `history/menu_history.db` | SQLite: история блюд + оценки + вес |

## Главный скилл

`/make-menu` — генерирует полное недельное меню. Запускай через `Skill tool`.

## Правила работы с БД

```bash
# Проверить блюда, которые не понравились
sqlite3 history/menu_history.db "SELECT DISTINCT dish_name FROM menu_dishes WHERE liked=0;"

# Посмотреть последние недели
sqlite3 history/menu_history.db "SELECT week_start, days_label, kcal_sergey FROM menu_weeks ORDER BY week_start DESC LIMIT 5;"

# Динамика веса
sqlite3 history/menu_history.db "SELECT person, weight_kg, recorded_at FROM weight_history ORDER BY person, recorded_at;"
```

## Когда пользователь говорит «обновить вес»

1. Обнови `profiles/sergey.json` или `profiles/elvira.json` → поле `weight_kg`
2. Добавь запись в `weight_history`:
```bash
sqlite3 history/menu_history.db "INSERT INTO weight_history (person, weight_kg, notes) VALUES ('Сергей', ВЕС, 'обновление');"
```
3. Скажи сколько потеряно с прошлого взвешивания.

## Когда пользователь говорит «не понравилось [блюдо]»

```bash
sqlite3 history/menu_history.db "UPDATE menu_dishes SET liked=0 WHERE dish_name LIKE '%НАЗВАНИЕ%';"
```
Подтверди: «Помечено как нежеланное, больше предлагать не буду».

## Структура HTML-меню

Эталон дизайна: `menus/2026/week_2026-04-22_vt-ch.html`

Карточка блюда содержит:
- **Лицевая сторона:** тип приёма, название, ккал, подсказка «↻ порции»
- **Модал (оборот):** список порций с граммовкой + рецепт (если блюдо требует готовки)

CSS-классы рецепта: `.recipe-divider`, `.recipe-label`, `.recipe-step`, `.recipe-num`, `.recipe-text`, `.recipe-note`

## Калорийность Сергея — динамический расчёт

```
BMR  = 10 × вес + 6.25 × 183 − 5 × возраст + 5
TDEE = BMR × 1.375
Цель = TDEE − 500   (не ниже 1800 ккал)
```

Возраст: родился 01.12.1991. Считать от текущего года.

## Ограничения Сергея — запомни навсегда

**НЕЛЬЗЯ:** сахар, белый хлеб/рис, жареное в масле, острые специи, копчёности, майонез, виноград/изюм/финики, кислые ягоды, грейпфрут, лимон, алкоголь, газировка, пакетированные соки.

**Порции:** белок ≤120 г за раз, гарнир 80–150 г, молоко ≤150 мл/день, кефир ≤200 мл/день, творог ≤120 г/день.

## Git и деплой

Репо: `https://github.com/colzphml/health_food.git`
GitHub Pages: `https://colzphml.github.io/health_food/`

После создания меню:
```bash
git add menus/ index.html history/menu_history.db
git commit -m "Меню на неделю [дата]"
git push origin main
```

**profiles/ и docs/ в .gitignore — никогда не пушить.**

# ARYA Dashboard

## 🔴 Открытые задачи по всем продуктам

```tasks
not done
```

## 📊 Статус продуктов

```dataview
TABLE статус AS "Статус", домен AS "Домен"
FROM "01-Products"
SORT file.name ASC
```

## 🔜 Ближайшие задачи

```tasks
not done
due before in 7 days
```

## 👥 Активные переговоры

```dataview
TABLE FROM "02-Partnership"
SORT file.name ASC
```


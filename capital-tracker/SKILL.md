---
name: capital-tracker
description: >
  Записывает пополнения и выводы капитала инвестора в Notion базу "Движение капитала".
  Используй когда пользователь пишет: пополнение, вывод, внёс, вывел, инвестор пополнил,
  новая транзакция, движение капитала, баланс инвестора.
  НИКОГДА не спрашивай куда записать. НИКОГДА не имитируй запись.
  ВСЕГДА используй инструмент terminal для выполнения curl команд.
---
# Движение капитала инвестора

## Правило
ОБЯЗАТЕЛЬНО используй инструмент **terminal** для каждого curl запроса.
НИКОГДА не имитируй выполнение. НИКОГДА не пиши результат без реального выполнения.

## Шаг 1 — Получить последний баланс (terminal)

Выполни через terminal:

```bash
export NOTION_KEY=$(grep 'NOTION_API_KEY=' ~/.hermes/.env | cut -d= -f2)
export CAPITAL_DB=$(grep 'CAPITAL_DB_ID=' ~/.hermes/.env | cut -d= -f2)
curl -s -X POST "https://api.notion.com/v1/databases/$CAPITAL_DB/query" \
  -H "Authorization: Bearer $NOTION_KEY" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  -d '{"sorts": [{"property": "Дата", "direction": "descending"}], "page_size": 1}' \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
results = data.get('results', [])
if results:
    bal = results[0]['properties'].get('Баланс после', {}).get('number')
    print(bal if bal is not None else 0)
else:
    print(0)
"
```

Сохрани результат как LAST_BALANCE.

## Шаг 2 — Записать транзакцию (terminal)

Подставь реальные значения из сообщения пользователя и выполни через terminal:

```bash
export NOTION_KEY=$(grep 'NOTION_API_KEY=' ~/.hermes/.env | cut -d= -f2)
export CAPITAL_DB=$(grep 'CAPITAL_DB_ID=' ~/.hermes/.env | cut -d= -f2)
TYPE="Пополнение"
AMOUNT=1000
INVESTOR="Иван"
DATE="2026-06-27"
NOTE=""
LAST_BALANCE=0

if [ "$TYPE" = "Пополнение" ]; then
  NEW_BALANCE=$(python3 -c "print($LAST_BALANCE + $AMOUNT)")
else
  NEW_BALANCE=$(python3 -c "print($LAST_BALANCE - $AMOUNT)")
fi
DATE_DISPLAY=$(date -d "$DATE" +%d.%m.%Y 2>/dev/null || echo "$DATE")
TITLE="$TYPE $DATE_DISPLAY"

curl -s -X POST https://api.notion.com/v1/pages \
  -H "Authorization: Bearer $NOTION_KEY" \
  -H "Content-Type: application/json" \
  -H "Notion-Version: 2022-06-28" \
  -d "{
    \"parent\": {\"database_id\": \"$CAPITAL_DB\"},
    \"properties\": {
      \"Название\": {\"title\": [{\"text\": {\"content\": \"$TITLE\"}}]},
      \"Дата\": {\"date\": {\"start\": \"$DATE\"}},
      \"Тип\": {\"select\": {\"name\": \"$TYPE\"}},
      \"Сумма USDT\": {\"number\": $AMOUNT},
      \"Инвестор\": {\"rich_text\": [{\"text\": {\"content\": \"$INVESTOR\"}}]},
      \"Баланс после\": {\"number\": $NEW_BALANCE},
      \"Заметка\": {\"rich_text\": [{\"text\": {\"content\": \"$NOTE\"}}]}
    }
  }"
```

## Переменные — подставь из сообщения пользователя
- TYPE — «Пополнение» или «Вывод»
- AMOUNT — сумма числом (1000, 500)
- INVESTOR — имя (если не указан — "")
- DATE — дата YYYY-MM-DD (если не указана — $(date +%Y-%m-%d))
- NOTE — заметка (если нет — "")

## Ответ после записи
✅ Записано в Движение капитала:
💰 Тип: $TYPE $AMOUNT USDT
👤 Инвестор: $INVESTOR
📅 Дата: $DATE_DISPLAY
📊 Баланс после: $NEW_BALANCE USDT

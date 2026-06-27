---
name: capital-tracker
description: >
  Записывает пополнения и выводы капитала инвестора в Notion базу "Движение капитала".
  Используй когда пользователь пишет: пополнение, вывод, внёс, вывел, инвестор пополнил,
  новая транзакция, движение капитала, баланс инвестора.
  НИКОГДА не спрашивай куда записать. НИКОГДА не имитируй запись.
  ВСЕГДА выполняй curl ниже с реальными данными.
---
# Движение капитала инвестора

## Правило
НИКОГДА не имитируй запись. ВСЕГДА выполняй curl ниже реально в терминале.

## Шаги
1. Извлеки из сообщения: тип (Пополнение/Вывод), сумму, имя инвестора, дату
2. Получи текущий баланс из Notion (curl запрос ниже)
3. Рассчитай новый баланс
4. Запиши в Notion через curl
5. Подтверди пользователю

## Шаг 1 — Получить последний баланс

```bash
export NOTION_KEY=$(grep 'NOTION_API_KEY=' ~/.hermes/.env | cut -d= -f2)
export CAPITAL_DB=$(grep 'CAPITAL_DB_ID=' ~/.hermes/.env | cut -d= -f2)

LAST_BALANCE=$(curl -s -X POST "https://api.notion.com/v1/databases/$CAPITAL_DB/query" \
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
")
echo "Текущий баланс: $LAST_BALANCE USDT"
```

## Шаг 2 — Записать транзакцию

```bash
export NOTION_KEY=$(grep 'NOTION_API_KEY=' ~/.hermes/.env | cut -d= -f2)
export CAPITAL_DB=$(grep 'CAPITAL_DB_ID=' ~/.hermes/.env | cut -d= -f2)

# Подставь из сообщения пользователя:
TYPE="Пополнение"       # Пополнение или Вывод
AMOUNT=1000             # сумма числом
INVESTOR="Иван"         # имя инвестора
DATE="2026-06-27"       # дата ISO формат YYYY-MM-DD
NOTE=""                 # заметка если есть
LAST_BALANCE=0          # результат из шага 1

# Рассчитай новый баланс
if [ "$TYPE" = "Пополнение" ]; then
  NEW_BALANCE=$(python3 -c "print($LAST_BALANCE + $AMOUNT)")
else
  NEW_BALANCE=$(python3 -c "print($LAST_BALANCE - $AMOUNT)")
fi

# Форматируй дату для названия DD.MM.YYYY
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
- AMOUNT — сумма числом без знака (1000, 500)
- INVESTOR — имя инвестора (если не указан — оставь пустым "")
- DATE — дата в формате YYYY-MM-DD (если не указана — используй сегодня: $(date +%Y-%m-%d))
- NOTE — любой дополнительный текст (если нет — "")

## Ответ после записи
✅ Записано в Движение капитала:
💰 $TYPE: $AMOUNT USDT
👤 Инвестор: $INVESTOR
📅 Дата: $DATE_DISPLAY
📊 Баланс после: $NEW_BALANCE USDT

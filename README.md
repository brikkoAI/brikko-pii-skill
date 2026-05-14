# Brikko PII Mask Skill

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![GitHub stars](https://img.shields.io/github/stars/brikkoAI/brikko-pii-skill?style=social)](https://github.com/brikkoAI/brikko-pii-skill/stargazers)
[![Works with: Claude Code](https://img.shields.io/badge/works%20with-Claude%20Code-D97757.svg)](https://docs.claude.com/en/docs/claude-code/overview)
[![Works with: Cursor](https://img.shields.io/badge/works%20with-Cursor-000000.svg)](https://cursor.com)
[![Works with: OpenClaw](https://img.shields.io/badge/works%20with-OpenClaw-4F46E5.svg)](https://github.com/openclaw/openclaw)
[![Works with: Codex](https://img.shields.io/badge/works%20with-Codex-10A37F.svg)](https://openai.com/codex)
[![152-ФЗ](https://img.shields.io/badge/152--ФЗ-compliant-success.svg)](#privacy)

> **Часть [Brikko Privacy Ecosystem](https://brikko.ru)** — open-source инфраструктура маскинга персональных данных перед AI для русского рынка.

Skill для AI-агентов (OpenClaw, Claude Code, Codex), который маскирует
персональные данные (ФИО, ИНН, СНИЛС, ОГРН, ОГРНИП, паспорт РФ, телефон,
email, банковский счёт) в тексте перед отправкой в LLM, и восстанавливает
плейсхолдеры в ответе.

> Если ты делаешь AI-приложения для бизнеса в РФ и не хочешь, чтобы реальные
> ИНН клиентов улетали в OpenAI/Anthropic — поставь этот skill, и агент
> автоматически защитит ПД.

## Что детектируется

— ФИО (с пониманием склонений: Иванов / Иванову / Иванова → один placeholder)
— ИНН 10 для юрлиц + 12 для физлиц (checksum-валидация по алгоритму ФНС)
— СНИЛС, ОГРН, ОГРНИП (checksum-валидация)
— Паспорт РФ (regex 4 цифры + 6 цифр)
— Телефон РФ (+7 999..., 8(495)..., 8 (812)...)
— Email
— Банковский счёт (20 цифр + контрольная цифра)
— IPv4

False-positive ratio < 1% за счёт checksum-валидации российских ID.

## Установка

### В OpenClaw

```bash
clawhub install brikko-pii-mask
# или
git clone https://github.com/brikkoAI/brikko-pii-skill.git \
  ~/.openclaw/skills/brikko-pii-mask
```

### В Claude Code (project-local)

```bash
git clone https://github.com/brikkoAI/brikko-pii-skill.git \
  .claude/skills/brikko-pii-mask
```

### В Claude Code (user-wide)

```bash
git clone https://github.com/brikkoAI/brikko-pii-skill.git \
  ~/.claude/skills/brikko-pii-mask
```

## Конфигурация

Skill использует Brikko Gateway API. Получи бесплатный ключ:

1. Регистрация: https://brikko.ru (200 ₽ welcome credit)
2. Создай ключ: brikko.ru → /app → «Создать API-ключ»
3. Экспортируй:

```bash
export BRIKKO_API_KEY=sk-brk-xxxxxxxxxxxx
```

**Self-hosted alternative:** установи Brikko Studio локально через
`npm install -g brikko-cli && brikko init` и укажи:

```bash
export BRIKKO_API_URL=http://localhost:3737
```

## Использование

```bash
# Шаг 1 — маскировать
echo "Клиент Иванов Иван, ИНН 7707083893, тел +7 999 123 45 67" \
  | python3 scripts/mask.py
# stdout: {"masked_text":"Клиент <NAME_1>, ИНН <INN_1>, тел <PHONE_1>",
#          "mapping_id":"abc123...", "count":3, "audit":[...]}

# Шаг 2 — отправить masked_text в любую LLM (Claude/GPT/Gemini)

# Шаг 3 — восстановить плейсхолдеры в ответе LLM
echo "Я отправил предложение <NAME_1> на номер <PHONE_1>." \
  | python3 scripts/restore.py --mapping-id abc123...
# stdout: Я отправил предложение Иванову Ивану на номер +7 999 123 45 67.
```

## Workflow для агента

```python
import json, subprocess

def with_pii_protection(prompt: str, llm_call) -> str:
    # 1. Mask
    r = json.loads(subprocess.check_output(
        ["python3", "scripts/mask.py"], input=prompt, text=True
    ))
    masked_prompt = r["masked_text"]

    # 2. LLM (any provider)
    response = llm_call(masked_prompt)

    # 3. Restore placeholders in response
    if r["mapping_id"]:
        response = subprocess.check_output(
            ["python3", "scripts/restore.py", "--mapping-id", r["mapping_id"]],
            input=response, text=True
        )

    return response
```

## Privacy

— Plaintext ПД ходят только до api.brikko.ru (HTTPS, TLS 1.3, LE certs)
— Mapping (placeholder → original) хранится в Redis api.brikko.ru с TTL 1 час
— Audit log Brikko содержит только категории и счётчики, **не plaintext**
— При self-hosted Studio данные не покидают локальную машину

## Цена

Маскирование бесплатно в M2 (promo для adoption).
В V3 — 0.01 ₽ за вызов (~10 ₽/день при 1000 вызовов).

## Поддержка

— Документация: https://brikko.ru/docs/skills/pii-mask
— Issues: https://github.com/brikkoAI/brikko-pii-skill/issues
— Email: hello@brikko.ru
— Telegram: [@brikko_ru](https://t.me/brikko_ru)

## Лицензия

MIT — см. [LICENSE](LICENSE).

---

## 🔗 Связанные продукты Brikko

| Артефакт | Установка | Аудитория |
|---|---|---|
| [brikko-studio](https://github.com/brikkoAI/brikko-studio) | `curl install.brikko.ru/studio.sh \| bash` | Desktop AI agent с MCP |
| [brikko-shield](https://github.com/brikkoAI/brikko-shield) | Chrome Web Store (скоро) | Маскинг в ChatGPT/Claude.ai |
| [brikko-cli](https://github.com/brikkoAI/brikko-cli) | `npm install -g brikko-cli` | CLI для Studio |
| **brikko-pii-skill** ★ (вы здесь) | `git clone` | Skill для Claude Code/Cursor |
| [n8n-nodes-brikko](https://github.com/brikkoAI/n8n-nodes-brikko) | `npm install n8n-nodes-brikko` | Маскинг в n8n workflows |
| [presidio-ru-recognizers](https://github.com/brikkoAI/presidio-ru-recognizers) | `pip install presidio-ru-recognizers` | Python recognizers для Presidio |

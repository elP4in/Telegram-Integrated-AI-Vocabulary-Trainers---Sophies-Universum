ABOUT CHATGPT

# Sophies Universum — Telegram Integrated AI Vocabulary Trainer

**Short description (for GitHub):** AI-powered vocabulary trainer integrated with Telegram — part of *Sophie’s Universum*, an adaptive language-learning project that helps users expand vocabulary through interactive chat-based exercises and personalized spaced repetition.

---

## Repository contents (scaffold)

This document contains a ready-to-copy repository scaffold. Files included as code blocks below:

* `README.md` (this file)
* `bot.py` (main Telegram bot)
* `db.py` (simple SQLite data layer)
* `sm2.py` (spaced repetition algorithm)
* `ai.py` (AI helpers: generate example sentences, distractors)
* `requirements.txt`
* `Dockerfile`
* `.gitignore`
* `LICENSE` (MIT)

---

## README (usage & setup)

````markdown
# Sophies Universum — Telegram AI Vocabulary Trainer

An adaptive vocabulary trainer that runs as a Telegram bot. Uses an AI backend (OpenAI-compatible) for generating contextual examples and plausible distractors, and an SM-2 based spaced repetition scheduler for reviews.

## Features
- Telegram native interaction (bot) — add the bot and start learning in chat.
- Add vocabulary items with translation and optional context sentence.
- Daily review sessions scheduled by SM-2 algorithm.
- AI-generated example sentences and multiple-choice distractors.
- Simple stats and progress tracking.

## Quickstart (development)

1. Create a Telegram bot via @BotFather and get `TELEGRAM_TOKEN`.
2. Obtain an AI API key (e.g. OpenAI) and set `OPENAI_API_KEY`.
3. Create a virtualenv and install dependencies:
   ```bash
   python -m venv .venv
   source .venv/bin/activate
   pip install -r requirements.txt
````

4. Initialize database (first run will auto-create `sophies.db`).
5. Run the bot:

   ```bash
   export TELEGRAM_TOKEN="<token>"
   export OPENAI_API_KEY="<key>"
   python bot.py
   ```

## Deployment

* Dockerfile provided for containerized deployment.
* For reliability, run under a process supervisor (systemd) or container orchestrator.

## Notes & Security

* Keep API keys and tokens secret. Use environment variables or a secrets manager in production.
* This scaffold is intentionally minimal — extend user management, authentication, analytics, and UX.

```
```

---

## bot.py

```python
"""bot.py
Main Telegram bot for Sophies Universum.
- Uses python-telegram-bot v13+ (or v20 async can be adapted)
- Minimal command set: /start, /add, /review, /stats
"""
import os
import logging
import sqlite3
from datetime import datetime
from telegram import Update, ReplyKeyboardMarkup, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Updater, CommandHandler, MessageHandler, Filters, CallbackContext, CallbackQueryHandler

from db import DB
from sm2 import schedule_next
from ai import generate_example_and_distractors

TELEGRAM_TOKEN = os.environ.get("TELEGRAM_TOKEN")
if not TELEGRAM_TOKEN:
    raise RuntimeError("Set TELEGRAM_TOKEN environment variable")

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

db = DB('sophies.db')

# Simple states (in-memory) for adding words — for production use proper FSM
pending_add = {}

def start(update: Update, context: CallbackContext):
    user = update.effective_user
    db.ensure_user(user.id, user.username)
    update.message.reply_text(
        f"Hi {user.first_name}! Welcome to Sophies Universum. Use /add to add words, /review to practice.")

def add_start(update: Update, context: CallbackContext):
    user_id = update.effective_user.id
    pending_add[user_id] = {'step': 1}
    update.message.reply_text("Send the word you want to learn (native or target).")

def message_handler(update: Update, context: CallbackContext):
    user_id = update.effective_user.id
    text = update.message.text.strip()
    if user_id in pending_add:
        state = pending_add[user_id]
        if state['step'] == 1:
            state['word'] = text
            state['step'] = 2
            update.message.reply_text("Send the translation (or translations, comma-separated).")
            return
        elif state['step'] == 2:
            translations = [t.strip() for t in text.split(',') if t.strip()]
            state['translations'] = translations
            state['step'] = 3
            update.message.reply_text("Optional: Send a short example sentence, or /skip to let AI generate one.")
            return
        elif state['step'] == 3:
            example = text
            db.add_card(user_id, state['word'], ','.join(state['translations']), example)
            update.message.reply_text(f"Added: {state['word']} -> {state['translations']}")
            del pending_add[user_id]
            return
    # fallback
    update.message.reply_text("I didn't understand — use /add to add new vocabulary or /review to start a session.")

def skip_example(update: Update, context: CallbackContext):
    user_id = update.effective_user.id
    if user_id not in pending_add:
        update.message.reply_text("Nothing to skip. Use /add to add a new word.")
        return
    state = pending_add[user_id]
    # call AI to generate example
    example, distractors = generate_example_and_distractors(state['word'], state.get('translations', []))
    db.add_card(user_id, state['word'], ','.join(state['translations']), example)
    update.message.reply_text(f"Added with AI example: {state['word']} -> {state['translations']}\nExample: {example}")
    del pending_add[user_id]

def review(update: Update, context: CallbackContext):
    user_id = update.effective_user.id
    due = db.get_due_cards(user_id, limit=10)
    if not due:
        update.message.reply_text("No cards due for review. Great job!")
        return
    # show first card as multiple choice
    card = due[0]
    example, distractors = generate_example_and_distractors(card['word'], card['translations'].split(','))
    options = [card['translations'].split(',')[0]] + distractors
    # shuffle options
    import random
    random.shuffle(options)
    keyboard = [InlineKeyboardButton(opt, callback_data=f"answer|{card['id']}|{opt}") for opt in options]
    markup = InlineKeyboardMarkup.from_row(keyboard)
    update.message.reply_text(f"Translate: {card['word']}\nExample: {example}", reply_markup=markup)

def button_handler(update: Update, context: CallbackContext):
    query = update.callback_query
    query.answer()
    data = query.data.split('|')
    if data[0] == 'answer':
        card_id = int(data[1])
        chosen = data[2]
        card = db.get_card(card_id)
        correct = card['translations'].split(',')[0]
        # simple quality score
        quality = 5 if chosen == correct else 2
        next_date = schedule_next(card['repetition'], card['interval'], card['easiness'], quality)
        db.update_after_review(card_id, correct if chosen==correct else chosen, quality, next_date)
        if chosen == correct:
            query.edit_message_text("✅ Correct!")
        else:
            query.edit_message_text(f"❌ Incorrect. Correct answer: {correct}")

def stats(update: Update, context: CallbackContext):
    user_id = update.effective_user.id
    s = db.stats(user_id)
    update.message.reply_text(f"Total cards: {s['total']} • Due: {s['due']}")


def main():
    updater = Updater(TELEGRAM_TOKEN)
    dp = updater.dispatcher
    dp.add_handler(CommandHandler('start', start))
    dp.add_handler(CommandHandler('add', add_start))
    dp.add_handler(CommandHandler('skip', skip_example))
    dp.add_handler(CommandHandler('review', review))
    dp.add_handler(CommandHandler('stats', stats))
    dp.add_handler(CallbackQueryHandler(button_handler))
    dp.add_handler(MessageHandler(Filters.text & ~Filters.command, message_handler))

    updater.start_polling()
    updater.idle()

if __name__ == '__main__':
    main()
```

---

## db.py

```python
"""db.py - tiny sqlite wrapper for cards and users
"""
import sqlite3
from datetime import datetime, timedelta

class DB:
    def __init__(self, path='sophies.db'):
        self.conn = sqlite3.connect(path, check_same_thread=False)
        self.conn.row_factory = sqlite3.Row
        self._ensure_tables()

    def _ensure_tables(self):
        cur = self.conn.cursor()
        cur.execute('''
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY,
            telegram_id INTEGER UNIQUE,
            username TEXT
        )
        ''')
        cur.execute('''
        CREATE TABLE IF NOT EXISTS cards (
            id INTEGER PRIMARY KEY,
            user_id INTEGER,
            word TEXT,
            translations TEXT,
            example TEXT,
            easiness REAL DEFAULT 2.5,
            repetition INTEGER DEFAULT 0,
            interval INTEGER DEFAULT 0,
            due_date TEXT
        )
        ''')
        self.conn.commit()

    def ensure_user(self, tg_id, username):
        cur = self.conn.cursor()
        cur.execute('INSERT OR IGNORE INTO users(telegram_id, username) VALUES (?,?)', (tg_id, username))
        self.conn.commit()

    def add_card(self, telegram_id, word, translations, example=None):
        # map telegram_id to internal user id
        cur = self.conn.cursor()
        cur.execute('SELECT id FROM users WHERE telegram_id=?', (telegram_id,))
        row = cur.fetchone()
        if not row:
            cur.execute('INSERT INTO users(telegram_id) VALUES (?)', (telegram_id,))
            self.conn.commit()
            user_id = cur.lastrowid
        else:
            user_id = row['id']
        due = datetime.utcnow().isoformat()
        cur.execute('INSERT INTO cards(user_id, word, translations, example, due_date) VALUES (?,?,?,?,?)',
                    (user_id, word, translations, example, due))
        self.conn.commit()

    def get_due_cards(self, telegram_id, limit=10):
        cur = self.conn.cursor()
        cur.execute('SELECT id FROM users WHERE telegram_id=?', (telegram_id,))
        row = cur.fetchone()
        if not row:
            return []
        user_id = row['id']
        cur.execute('SELECT * FROM cards WHERE user_id=? AND (due_date <= ?) ORDER BY due_date LIMIT ?',
                    (user_id, datetime.utcnow().isoformat(), limit))
        rows = cur.fetchall()
        return [dict(r) for r in rows]

    def get_card(self, card_id):
        cur = self.conn.cursor()
        cur.execute('SELECT * FROM cards WHERE id=?', (card_id,))
        row = cur.fetchone()
        return dict(row) if row else None

    def update_after_review(self, card_id, answer, quality, next_date_iso):
        cur = self.conn.cursor()
        # load current
        cur.execute('SELECT * FROM cards WHERE id=?', (card_id,))
        row = cur.fetchone()
        if not row:
            return
        repetition = row['repetition'] + 1 if quality >= 3 else 0
        interval = (row['interval'] if repetition==0 else int((datetime.fromisoformat(next_date_iso) - datetime.utcnow()).days))
        easiness = row['easiness']
        cur.execute('UPDATE cards SET repetition=?, interval=?, easiness=?, due_date=? WHERE id=?',
                    (repetition, interval, easiness, next_date_iso, card_id))
        self.conn.commit()

    def stats(self, telegram_id):
        cur = self.conn.cursor()
        cur.execute('SELECT id FROM users WHERE telegram_id=?', (telegram_id,))
        row = cur.fetchone()
        if not row:
            return {'total':0,'due':0}
        user_id = row['id']
        cur.execute('SELECT COUNT(*) as c FROM cards WHERE user_id=?', (user_id,))
        total = cur.fetchone()['c']
        cur.execute('SELECT COUNT(*) as c FROM cards WHERE user_id=? AND due_date <= ?', (user_id, datetime.utcnow().isoformat()))
        due = cur.fetchone()['c']
        return {'total': total, 'due': due}
```

---

## sm2.py

```python
"""Simple SM-2 implementation
"""
from datetime import datetime, timedelta

def schedule_next(repetition, interval, easiness, quality):
    # repetition: integer
    # interval: previous interval in days
    # easiness: float
    # quality: int 0-5
    if quality < 3:
        repetition = 0
        interval = 1
    else:
        if repetition == 0:
            interval = 1
        elif repetition == 1:
            interval = 6
        else:
            interval = int(interval * easiness)
        repetition += 1
    # update easiness
    easiness = max(1.3, easiness + (0.1 - (5 - quality) * (0.08 + (5 - quality) * 0.02)))
    next_date = datetime.utcnow() + timedelta(days=interval)
    # return next_date iso string — db will store it
    return next_date.isoformat()
```

---

## ai.py

```python
"""ai.py - thin wrapper to call AI provider for examples/distractors
This file contains placeholders. Replace with a real integration (OpenAI, Anthropic, etc.)
"""
import os

OPENAI_API_KEY = os.environ.get('OPENAI_API_KEY')

# Minimal fallback if no AI key is set

def generate_example_and_distractors(word, translations):
    """Return (example_sentence, [distractor1, distractor2])"""
    if not OPENAI_API_KEY:
        # simple heuristic fallback
        example = f"I used the word {word} in a sentence to remember it."
        distractors = [f"{translations[0]}typo" if translations else 'foo', 'bar']
        return example, distractors
    # If key exists, implement a call to OpenAI or another provider here.
    # Example pseudocode:
    # prompt = f"Provide a short example sentence using the word '{word}' and two plausible wrong translations (one-word each).\nSentence: ...\nWrong translations: x, y"
    # call API and parse response
    # return sentence, [x,y]
    example = f"AI-generated example for {word} (stub)"
    distractors = ['distractor1', 'distractor2']
    return example, distractors
```

---

## requirements.txt

```
python-telegram-bot==13.14
requests
# add openai or other client if desired
```

---

## Dockerfile

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY . /app
RUN pip install --no-cache-dir -r requirements.txt
ENV PYTHONUNBUFFERED=1
CMD ["python","bot.py"]
```

---

## .gitignore

```
__pycache__/
*.pyc
.sophies.env
sophies.db
.env
.venv/
```

---

## LICENSE (MIT)

```text
MIT License

Copyright (c) 2025

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

[...standard MIT text truncated for brevity in this scaffold...]
```

---

### Next steps (suggested)

* Copy the files into a new GitHub repository named `sophies-universum`.
* Implement the AI provider calls in `ai.py` (OpenAI/other) and safely store API keys.
* Improve UX: add conversation states (use `ConversationHandler` or FSM), localization, and richer review flows (audio, images).
* Add tests and CI.

---

If you'd like, I can:

* export these files individually as downloadable files,
* generate a `README.md` file in the repo format,
* or convert the bot to use `python-telegram-bot` async v20 or to a Node.js implementation.

Tell me which next step you want and I will proceed.

# 🤖 Ref Desk — AI-Powered FAQ Chatbot Using NLP

A Final Year Project: a chatbot that understands user questions using
NLP (tokenization, stopword removal, lemmatization, TF-IDF, cosine
similarity) and matches them against a managed FAQ knowledge base.

---

## 1. Features

### Basic
- ✅ FAQ Matching (TF-IDF + Cosine Similarity)
- ✅ Chat Interface (Bootstrap, custom theme, fully responsive)
- ✅ Instant Responses
- ✅ Search Functionality (suggestion chips pulled from live FAQ data)

### Advanced
- ✅ Voice Input (Web Speech API — speech-to-text)
- ✅ Speech Output (Web Speech API — text-to-speech, toggle button)
- ✅ Chat History (persisted per-session in the database, restored on reload)
- ✅ Admin Panel (login, dashboard analytics, full FAQ CRUD)
- ✅ Dark Mode (toggle, remembers preference for the browser session)
- ⚙️ Multi-Language Support — UI is structured to support it (see
  *Future Enhancements*); voice input already accepts other languages
  by changing `recognition.lang` in `chat.js`, but FAQ matching itself
  is English-tuned out of the box.

### Confidence Transparency (signature feature)
Every bot reply shows a **match %** bar — the actual cosine similarity
score — so the project visibly demonstrates the underlying NLP math,
not just a black-box answer.

---

## 2. Architecture (MVC)

```
faq_chatbot/
├── run.py                          # Entry point
├── requirements.txt
├── database/
│   └── schema.sql                  # Full DB schema + seed data
├── app/
│   ├── __init__.py                 # Flask app factory
│   ├── models/                     # MODEL layer
│   │   ├── database.py             # Connection + init/seed logic
│   │   ├── faq_model.py            # FAQ CRUD queries
│   │   ├── chat_model.py           # Chat history + analytics queries
│   │   └── nlp_engine.py           # Preprocessing + TF-IDF + cosine similarity
│   ├── controllers/                # CONTROLLER layer
│   │   ├── chat_controller.py      # /  and /api/chat, /api/history
│   │   └── admin_controller.py     # /admin/* (login, dashboard, FAQ CRUD)
│   ├── templates/                  # VIEW layer
│   │   ├── index.html              # Chat UI
│   │   └── admin/
│   │       ├── base.html
│   │       ├── login.html
│   │       ├── dashboard.html
│   │       └── faqs.html
│   └── static/
│       ├── css/ (style.css, admin.css)
│       └── js/  (chat.js)
└── docs/
    └── PROJECT_REPORT.md            # Extended write-up for submission
```

**Why this is MVC:**
- **Model** — `app/models/` owns every SQL query and the NLP logic. Controllers never write raw SQL.
- **View** — `app/templates/` is pure presentation (Jinja2 + Bootstrap), no business logic.
- **Controller** — `app/controllers/` (Flask Blueprints) wires HTTP requests to model calls and chooses which view to render.

---

## 3. How the NLP Matching Works

```
User query
   │
   ▼
preprocess_text()
   ├─ lowercase + strip punctuation
   ├─ tokenize            (nltk.word_tokenize)
   ├─ remove stopwords     (nltk.corpus.stopwords)
   └─ lemmatize each token (nltk.stem.WordNetLemmatizer)
   │
   ▼
TfidfVectorizer.transform()   →  query vector
   │
   ▼
cosine_similarity(query_vector, faq_vectors)  →  score per FAQ
   │
   ▼
argmax(scores)
   ├─ score ≥ 0.20  →  return that FAQ's answer + score
   └─ score <  0.20  →  "Sorry, I could not find a suitable answer."
```

The `0.20` confidence threshold lives in `app/models/nlp_engine.py` as
`CONFIDENCE_THRESHOLD` — tune it up for stricter matching, or down if
the bot is rejecting valid questions too often.

Every time an admin adds, edits, deletes, or deactivates an FAQ, the
TF-IDF vector space is **rebuilt immediately** (`refresh_nlp_engine()`
in `admin_controller.py`), so the chatbot is always matching against
the current dataset with no restart required.

---

## 4. Installation Guide

### Prerequisites
- Python 3.10+
- pip

### Step-by-step

```bash
# 1. Clone / unzip the project, then enter the folder
cd faq_chatbot

# 2. (Recommended) Create a virtual environment
python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Download required NLTK data (one-time)
python3 -c "import nltk; nltk.download('punkt'); nltk.download('punkt_tab'); nltk.download('stopwords'); nltk.download('wordnet'); nltk.download('omw-1.4')"

# 5. Run the app (this also creates and seeds the database automatically)
python3 run.py
```

Open **http://127.0.0.1:5000** in your browser.

### Admin Panel
- URL: **http://127.0.0.1:5000/admin/login**
- Default credentials: `admin` / `admin123`
- ⚠️ Change this password before any real deployment (see Security Notes below).

---

## 5. Database

Ships with **SQLite** by default (`database/faq_chatbot.db`, created
automatically on first run — zero configuration). The schema in
`database/schema.sql` is written in portable SQL and documented for
**MySQL** migration:

### To migrate to MySQL
1. Install `mysql-connector-python` or `PyMySQL`.
2. In `database/schema.sql`, change:
   - `INTEGER PRIMARY KEY AUTOINCREMENT` → `INT AUTO_INCREMENT PRIMARY KEY`
   - `INTEGER` (boolean flags) → `TINYINT(1)`
3. Replace `app/models/database.py`'s `get_db_connection()` to use
   your MySQL connector instead of `sqlite3.connect()`. Because all
   queries in `faq_model.py` / `chat_model.py` use plain `?`
   placeholders and standard SQL, only the connection layer needs to
   change — no query rewrites required for basic CRUD (MySQL uses
   `%s` placeholders, so a small adapter shim is the only extra step).

---

## 6. Tech Stack

| Layer      | Technology |
|------------|------------|
| Frontend   | HTML5, CSS3, JavaScript (ES6), Bootstrap 5 |
| Backend    | Python 3, Flask, Flask Blueprints |
| NLP        | NLTK (tokenize, stopwords, lemmatize), Scikit-Learn (TF-IDF, cosine similarity) |
| Database   | SQLite (default) / MySQL-compatible schema |
| Auth       | Werkzeug password hashing + Flask sessions |
| Voice      | Web Speech API (`SpeechRecognition`, `SpeechSynthesisUtterance`) |

---

## 7. Modules Recap

**User Module** — Ask Question, View Answers, Voice Input/Output, Dark Mode, Chat History
**NLP Engine** — Text Preprocessing, TF-IDF Vectorization, Cosine Similarity Matching
**Admin Module** — Login, Dashboard (analytics + unanswered-query log), Add / Edit / Delete / Activate-Deactivate FAQs

---

## 8. Future Enhancements

- **OpenAI / LLM API Integration** — fall back to a generative model when confidence is low, instead of the static "could not find an answer" message.
- **Sentiment Analysis** — flag frustrated users for human handoff.
- **Learning from User Queries** — the `unanswered_queries` table already logs every low-confidence question; a future admin feature could one-click promote these into new FAQs.
- **True Multi-Language Support** — swap `TfidfVectorizer`'s English stopword list per detected language, or use multilingual embeddings (e.g. `sentence-transformers`) instead of TF-IDF.
- **WhatsApp Integration** — wrap `/api/chat` behind the WhatsApp Business Cloud API webhook.

---

## 9. Security Notes (for production use)

- Change the default admin password immediately (`admin123` is a placeholder for grading/demo purposes).
- Set a strong `FAQ_CHATBOT_SECRET_KEY` environment variable instead of the default dev key in `app/__init__.py`.
- Set `debug=False` in `run.py` before deploying anywhere public.
- Use a production WSGI server (e.g. Gunicorn) instead of Flask's built-in dev server.

---

## 10. Tested Functionality

All of the following were verified against a live running instance during development:
- ✅ Homepage renders (HTTP 200), all static assets load
- ✅ Paraphrased questions correctly matched to the right FAQ with a sensible confidence score
- ✅ Unrelated/out-of-scope questions correctly trigger the fallback message
- ✅ Chat history is persisted per session and restored on page reload
- ✅ Admin login/logout and route protection (unauthenticated users are redirected)
- ✅ Adding an FAQ via the admin panel makes it **immediately** matchable by the chatbot (no restart)
- ✅ Deactivating an FAQ removes it from chatbot matching while keeping it in the admin list
- ✅ Deleting an FAQ that already has chat history works correctly (history is preserved with the reference cleared, not blocked by a foreign-key error)

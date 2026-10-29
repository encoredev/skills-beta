# Cron jobs in Python

Import: `from encore import cron`.

```python
from encore import cron
from fastapi import FastAPI

app = FastAPI()

_ = cron.Job(
    "welcome-email",                  # kebab-case, unique
    path="/internal/welcome-emails",
    title="Send welcome emails",      # optional
    every="2h",                       # or schedule="0 4 15 * *" (UTC). Exactly one.
)

@app.post("/internal/welcome-emails")
async def send_welcome_emails() -> None:
    ...
```

- Assign it to a module-level variable (`_` is fine). A bare `cron.Job(...)` expression, or one inside a function, `if` or `try`, is a parse error.
- `cron.Job` raises `ValueError` at import if both or neither of `every`/`schedule` is set, or `path` doesn't start with `/`.
- Django and Flask views work the same; they must accept `POST`.

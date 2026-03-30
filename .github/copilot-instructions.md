---
description: >-
  Workspace instructions for Soc Ops - a Social Bingo game built with FastAPI, 
  Jinja2, and HTMX. Contains project conventions, dev environment setup, 
  and key architecture patterns.
---

# Soc Ops — Copilot Workspace Instructions

**Soc Ops** is a Social Bingo game for in-person mixers. This is an educational project that demonstrates [Copilot-driven development workflows](workshop/00-overview.md).

## Quick Start

```bash
# Install dependencies (one-time)
uv sync

# Run dev server (auto-reload on file changes)
uv run uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# Run tests
uv run pytest

# Lint code
uv run ruff check .
```

## Stack

| Layer | Technology |
|-------|-----------|
| **Framework** | FastAPI (Python 3.13+) |
| **Templates** | Jinja2 |
| **Interactivity** | HTMX (no page reloads) |
| **Styling** | Custom CSS utilities (Tailwind-like) |
| **Session Management** | Signed cookies (itsdangerous) |
| **Testing** | pytest + httpx |

## Architecture

### Directory Structure
```
app/
├── main.py              # FastAPI app, routes, session middleware
├── models.py            # Pydantic models (GameState, BingoSquareData, BingoLine)
├── game_logic.py        # Pure functions: board generation, bingo detection
├── game_service.py      # GameSession dataclass, session store
├── data.py              # QUESTIONS list and FREE_SPACE constant
├── templates/           # Jinja2 templates
│   ├── base.html        # Base layout with HTMX script
│   ├── home.html        # Initial landing page
│   └── components/
│       ├── start_screen.html    # "Start Game" button
│       ├── game_screen.html     # Active bingo board + modal
│       ├── bingo_board.html     # 5x5 grid of clickable squares
│       └── bingo_modal.html     # Win celebration modal
└── static/
    ├── css/app.css      # Custom utility classes
    └── js/htmx.min.js   # HTMX library

tests/
├── test_api.py          # FastAPI endpoint tests (8 tests)
└── test_game_logic.py   # Game logic unit tests (17 tests)
```

### State Management Pattern

**Server-side session state** (`GameSession`):
- `game_state`: START → PLAYING → BINGO
- `board`: List of 25 `BingoSquareData` objects
- `winning_line`: Set when bingo detected
- `show_bingo_modal`: Controls modal visibility

**Persistence**: Signed cookies with `SessionMiddleware` + in-memory `_sessions` dict

**Updates**: HTMX `hx-post` triggers route, returns HTML fragment that replaces page section

### Game Logic

Template for adding features:

```python
# game_logic.py: Pure functions (no state)
def new_game_feature(board: list[BingoSquareData]) -> SomeResult:
    """Testable logic without side effects."""

# game_service.py: Add method to GameSession
def trigger_feature(self) -> None:
    result = new_game_feature(self.board)
    # Update state...

# main.py: Route returns updated template
@app.post("/endpoint")
async def endpoint(request: Request) -> Response:
    session = _get_game_session(request)
    session.trigger_feature()
    return templates.TemplateResponse(...)
```

## Key Conventions

### Code Style
- **Python**: snake_case, type hints required, 88-char line length
- **Linting**: `ruff` (E, F, I, N, W rules)
- **Templates**: Minimal logic, use filters/tests from Jinja2
- **CSS**: Utility classes, CSS variables for theme colors

### Naming
- Game routes: `/start`, `/toggle/{square_id}`, `/reset`, `/dismiss-modal`
- Template components: Named by feature (`game_screen.html`, `bingo_board.html`)
- Models: Domain-focused (GameState, BingoSquareData, not DtoX)

### Template Patterns
```html
<!-- HTMX endpoint returning partial (not full page) -->
<button hx-post="/toggle/{{ square.id }}" hx-target="main" hx-swap="innerHTML">
  {{ square.text }}
</button>

<!-- Conditional rendering -->
{% if session.has_bingo %}
  <div class="modal">You won!</div>
{% endif %}

<!-- Iteration with state -->
{% for square in session.board %}
  {% if square.is_marked %}<span class="marked">✓</span>{% endif %}
{% endfor %}
```

## Related Documentation

- **Setup & Environment**: [setup.prompt.md](prompts/setup.prompt.md)
- **CSS Utilities**: [css-utilities.instructions.md](instructions/css-utilities.instructions.md)
- **Frontend Design**: [frontend-design.instructions.md](instructions/frontend-design.instructions.md)
- **Dev Workflow**: [CONTRIBUTING.md](CONTRIBUTING.md)
- **Workshop Guide**: [workshop/](workshop/) (offline lab materials)

## Testing Checklist

Before committing:
- [ ] `uv run pytest` passes (25 tests)
- [ ] `uv run ruff check .` passes (no errors)
- [ ] New features have corresponding tests
- [ ] No unused imports or variables

## Common Tasks

### Add a New Game Feature
1. Implement pure function in `game_logic.py` with tests in `test_game_logic.py`
2. Add method to `GameSession` in `game_service.py`
3. Create POST route in `main.py`
4. Build/update template component in `templates/components/`
5. Wire up HTMX trigger (usually a button or form)

### Modify Board Appearance
- Edit `templates/components/bingo_board.html` (structure)
- Update utilities in `app/static/css/app.css` or `base.html` `<style>` tag
- Refer to [css-utilities.instructions.md](instructions/css-utilities.instructions.md) for available classes

### Update Question Bank
- Edit `app/data.py` - modify `QUESTIONS` list
- Tests auto-verify board generation consumes 24 unique questions
- Central theme: SOC (Security Operations Center) tasks and challenges

### Add an API Endpoint
1. Create route in `main.py` with `@app.get()` or `@app.post()`
2. Use `_get_game_session(request)` to access state
3. Return `TemplateResponse` with updated component name
4. Add test to `tests/test_api.py` using `TestClient`

## Dev Environment Notes

- **Python 3.13+ required** (type hints, walrus operators, etc.)
- **uv** package manager (handles venv + deps automatically)
- **Hot reload**: Uvicorn watches `app/` and `templates/` directories
- **No browser extension required**: Plain HTML + HTMX, works in any browser
- **IMPORTANT**: Do NOT use VS Code Simple Browser (HTMX requires full browser support)

## Potential Pitfalls

| Issue | Solution |
|-------|----------|
| Port 8000 already in use | Change `--port` in run command or kill existing process (`lsof -i :8000`) |
| Changes not reloading | Ensure Uvicorn is running with `--reload` flag |
| HTMX not working | Open in real browser, not Simple Browser; check network tab for errors |
| Session lost after restart | Sessions are in-memory; restart clears games (expected behavior) |
| Import errors when running pytest | Ensure `uv sync` was run; check `pyproject.toml` dependencies |
| CSS changes not visible | Clear browser cache or do hard refresh (Ctrl+Shift+R) |

## Example: "Add a New Route"

```python
# 1. game_logic.py
def get_hint(board: list[BingoSquareData]) -> str:
    """Return text of first unmarked square."""
    for square in board:
        if not square.is_marked and not square.is_free_space:
            return square.text
    return "No hints available"

# 2. test_game_logic.py
def test_get_hint():
    board = generate_board()
    hint = get_hint(board)
    assert len(hint) > 0

# 3. game_service.py (GameSession class)
def reveal_hint(self) -> None:
    self.hint = get_hint(self.board)

# 4. main.py
@app.post("/hint", response_class=HTMLResponse)
async def show_hint(request: Request) -> Response:
    session = _get_game_session(request)
    session.reveal_hint()
    return templates.TemplateResponse(
        request, "components/hint_display.html", {"hint": session.hint}
    )

# 5. game_screen.html
<button hx-post="/hint" hx-target="#hint-box" hx-swap="innerHTML">
  Get Hint
</button>
<div id="hint-box"></div>

# 6. templates/components/hint_display.html
<p class="text-sm italic">{{ hint }}</p>
```

---

**Last Updated**: 2026-03  
**Python Version**: 3.13+  
**uv Version**: 0.11+

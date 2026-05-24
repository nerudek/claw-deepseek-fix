# DeepSeek V4 Fix for Claw Code

> **Problem:** Claw Code (fork Claude Code) crashuje z błędem 400 Bad Request przy użyciu DeepSeek V4 API.
> 
> **Rozwiązanie:** Patch do Claw Code który zachowuje `reasoning_content` i pomija puste `tool_calls`.

---

## Problemy

### 1. `tool_calls: []` — pusta tablica

DeepSeek API (i inni "strict" providerzy jak NVIDIA NIM) odrzucają wiadomości asystenta z pustą tablicą `tool_calls`:

```
400 Bad Request: Invalid 'messages[N].tool_calls': empty array.
Expected an array with minimum length 1, but got an empty array instead.
```

**Kiedy występuje:** Gdy model odpowiada tekstem bez wywołania narzędzi (tool calls).

### 2. `reasoning_content` — brakujące pole

DeepSeek V4 zwraca `reasoning_content` w każdej odpowiedzi (tryb "thinking"). To pole MUSI być przekazane z powrotem do API w kolejnej turze:

```
400 Bad Request: The reasoning_content in the thinking mode must be passed back to the API.
```

**Kiedy występuje:** Na drugiej turze rozmowy lub po tool callu, gdy historia nie zawiera `reasoning_content`.

---

## Rozwiązanie

### Commit 1: `fix(api): omit empty tool_calls array`

**Plik:** `rust/crates/api/src/providers/openai_compat.rs`

Zmienia `translate_message()` — nie wysyła pola `tool_calls` gdy tablica jest pusta:

```rust
// BEFORE: zawsze wysyłał tool_calls: []
// AFTER: wysyła tylko gdy niepusty
if !tool_calls.is_empty() {
    msg.insert("tool_calls", json!(tool_calls));
}
```

### Commit 2: `fix(deepseek): preserve reasoning_content`

**Pliki:**
- `rust/crates/api/src/types.rs` — nowe typy `ReasoningContent`
- `rust/crates/api/src/providers/openai_compat.rs` — capture + passthrough
- `rust/crates/runtime/src/conversation.rs` — `AssistantEvent::Reasoning`
- `rust/crates/runtime/src/session.rs` — `ContentBlock::Reasoning`
- `rust/crates/tools/src/lib.rs` — obsługa bloków
- `rust/crates/rusty-claude-cli/src/main.rs` — konwersja event → block

Zmienia logikę — zawsze wysyła `reasoning_content` (nawet pusty `""`) dla wiadomości asystenta:

```rust
// DeepSeek V4 requires reasoning_content to be passed back
msg.insert("reasoning_content", json!(reasoning_content));
```

---

## Instalacja

### Wymagania
- Rust 1.70+
- Claw Code (source)

### Krok 1: Sklonuj repo z fixem

```bash
git clone https://github.com/nerudek/claw-deepseek-fix.git
cd claw-deepseek-fix
```

### Krok 2: Skompiluj

```bash
cd rust
cargo build --release
```

### Krok 3: Podmień binarkę

```bash
# Backup starej wersji
cp ~/claw-code-local/rust/target/release/claw ~/claw-backup

# Kopiuj nową
cp target/release/claw ~/claw-code-local/rust/target/release/claw
```

### Krok 4: Skonfiguruj DeepSeek API

```bash
export OPENAI_API_KEY="sk-..."
export OPENAI_BASE_URL="https://api.deepseek.com/v1"
```

### Krok 5: Test

```bash
claw --model deepseek-v4-pro
> wylistuj pliki
```

---

## Testowane modele

| Model | Status | Uwagi |
|-------|--------|-------|
| `deepseek-v4-pro` | ✅ Działa | Pełne wsparcie reasoning |
| `deepseek-v4-flash` | ✅ Działa | Szybszy, tańszy |
| `deepseek-chat` | ✅ Działa | V3, brak reasoning — też działa |
| `deepseek-reasoner` | ✅ Działa | Legacy, inny format |

---

## Znane ograniczenia

- Fix jest specyficzny dla Claw Code (fork Claude Code)
- Nie testowano z Claude Code oryginalnym (może wymagać adaptacji)
- Inni providerzy OpenAI-compatible mogą ignorować `reasoning_content`

---

## Autorzy

- **YOLO** (Kimi M4) — discovery, research, initial patch
- **Claude M4** (Sage) — runtime fixes, session handling
- **Tomek** — testing, deployment

## Licencja

MIT — ten patch jest public domain. Używaj jak chcesz.

---

## Linki

- **GitHub:** https://github.com/nerudek/claw-deepseek-fix
- **GitLab:** https://gitlab.com/nerudek/claw-deepseek-fix *(mirror)*
- **Strona:** https://nerudek.github.io/claw-deepseek-fix
- **Upstream issue:** https://github.com/ultraworkers/claw-code/issues/2821
- **Related:** https://github.com/zeroclaw-labs/zeroclaw/issues/6298

---

If this saved you time: [☕ PayPal.me/nerudek](https://www.paypal.me/nerudek)

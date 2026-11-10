# Skills

Własne skille w standardzie Agent Skills, instalowane przez [Vercel Skills CLI](https://github.com/vercel-labs/skills).
Repozytorium nie wymaga publikacji pakietu npm. `npx` uruchamia instalator `skills`,
który pobiera wskazane katalogi z Git lub lokalnego checkoutu.

## Instalacja

Wymagania skilla `team`: Node.js 18+, Git, Linux/macOS i powłoka POSIX do wykonywania
sprawdzanych poleceń. Wszystkie sesje zespołu muszą widzieć ten sam lokalny katalog
repozytorium. Natywny Windows oraz współdzielone dyski sieciowe nie są zweryfikowane.

Po opublikowaniu zawartości tego repozytorium:

```sh
npx skills add mrstroz/skills --skill team --global --agent claude-code codex opencode
```

Już teraz, z katalogu tego checkoutu:

```sh
npx skills add . --list
npx skills add . --skill team --global --agent claude-code codex opencode
```

`--global` oznacza instalację dla użytkownika, dostępną we wszystkich jego projektach.
Bez tej flagi instalacja jest projektowa. Możesz podać inne agenty obsługiwane przez
CLI, np. `gemini-cli` lub `github-copilot`. Dla instalacji bez pytań dodaj `--yes`.
Domyślne dowiązania współdzielą zainstalowaną kopię; `--copy` tworzy osobne kopie.
Instalacja z lokalnej ścieżki nie oznacza automatycznej synchronizacji każdej edycji
checkoutu — po zmianach ponów `add`.

Przy instalacji globalnej dla tych trzech agentów (sprawdzone na `skills@1.7.6`) powstaje
jedna kopia i jedno dowiązanie:

| Ścieżka | Co to jest | Kto z niej korzysta |
|---|---|---|
| `~/.agents/skills/team` | właściwa kopia skilla | Codex, OpenCode |
| `~/.claude/skills/team` | dowiązanie do kopii powyżej | Claude Code |

Instalator nie zapisuje niczego w `~/.config/opencode/skills/`. Po instalacji sprawdź
w sesji OpenCode, czy skill `team` jest widoczny — tego nie zweryfikowaliśmy.

Repozytorium prywatne wymaga skonfigurowanego dostępu Git/SSH. Można podać URL SSH:

```sh
npx skills add git@github.com:mrstroz/skills.git --skill team -g -a claude-code codex opencode
```

## Zarządzanie

```sh
npx skills list --global
npx skills check
npx skills update
npx skills remove team --global --agent claude-code codex opencode
```

Sprawdź zakres zmian pokazywany przez `update`; dostępne opcje zależą od wersji CLI.
Ponowny `add` z tym samym źródłem jest również sposobem odświeżenia skilla.
Testowaną wersję instalatora można przypiąć jako `npx skills@1.7.6 ...`.
Po instalacji sprawdź katalog skilli w CLI lub uruchom nową sesję.

Usunięcie skilla nie usuwa historii `.team/` w projektach i nie zamyka działających
sesji. Najpierw zakończ run przez `close`. Run zachowuje
własną kopię skryptu i instrukcji, więc aktualizacja instalacji nie zmienia trwającej pracy.
`gc --dry-run` pokazuje stare referencje Git możliwe do usunięcia, a `gc` usuwa je tylko
dla zamkniętych runów; dzienniki zostają. Nie usuwaj aktywnej `.team/`.

## Team

Samodzielny [skill team](skills/team/SKILL.md) koordynuje trzy role:

- **driver** wykonuje zadanie, utrzymuje plan i przedstawia wynik;
- **architect** ocenia podejście i interfejsy;
- **tester** niezależnie sprawdza rezultat.

W Claude Code zacznij np. `/team driver: dodaj eksport CSV`. W Codexie:
`$team jako driver: dodaj eksport CSV`. W OpenCode poproś o załadowanie skilla `team`
i podaj rolę oraz zadanie. Przy dołączaniu podaj rolę i ścieżkę runu wypisaną przez drivera.
Nie używamy już nazw pluginowych `/team:driver`, `/team:architect`, `/team:tester`.
Skill jest uruchamiany jawnie w Claude Code i Codexie; inne hosty mogą inaczej traktować
metadane aktywacji.

Sesje mogą używać różnych CLI, ale skill sam ich nie uruchamia. Protokół działa lokalnie,
bez serwera MCP, konta pośrednika i dodatkowych zależności npm. Wiadomości i dowody testów
trafiają do `.team/<run>/`; skrypt tworzy również snapshoty pod `refs/team/` w Git.
Katalog `.team/` jest wyłączany lokalnie ze śledzenia w Git.

### Komunikacja

Skill nie budzi bezczynnych sesji. Wiadomości czekają w dzienniku runu, a sesję, która
ma coś do przeczytania, wznawia użytkownik nadzorujący pracę.

| Sesja | Jak odbiera wiadomości |
|---|---|
| Claude Code | `wait` jako zadanie w tle; jego zakończenie rozpoczyna kolejną turę |
| Codex, OpenCode, inne CLI | czyta po wznowieniu przez użytkownika (`wait --timeout 0`); w trakcie tury może krótko poczekać |

Po każdym `post` skrypt wypisuje `not waiting: <id>` dla adresatów bez działającego `wait`.
Driver przekazuje to użytkownikowi — to lista zakładek do wznowienia. `status` pokazuje,
kto ma nieprzeczytane wiadomości. Szczegóły: sekcja „Waiting” w
[protocol.md](skills/team/references/protocol.md).

### Polecenia pomocnicze

Wstaw absolutną ścieżkę zainstalowanego skilla lub kopii w runie:

```sh
node /path/to/team/scripts/team.mjs help
node /path/to/team/scripts/team.mjs doctor
node /path/to/team/scripts/team.mjs runs
node /path/to/run/team.mjs status
node /path/to/run/team.mjs gates
```

`doctor` sprawdza możliwość zapisu i snapshotów w bieżącym repozytorium.

## Rozwój i pochodzenie

```sh
npm run check
npm test
```

Testy uruchamiają prawdziwe procesy skryptu w tymczasowych repozytoriach i nie wysyłają
promptów do modeli. Instalację globalną sprawdzono przez `skills@1.7.6` w odizolowanym
katalogu domowym; odinstalowania i prawdziwego runu z kilkoma sesjami jeszcze nie wykonano.

Kod pochodzi z `mrstroz/claude-code-plugins`, `team` na gałęzi `feat/team-plugin`, commit
`55a0e27`. Został przeniesiony do samodzielnego katalogu; stare repozytorium pozostaje
bez zmian, aby umożliwić porównanie i dotychczasowe instalacje pluginu. Ta wersja nie ma
watchera, polecenia `bind` ani adapterów `codex queue` / `opencode api`; ich kod jest
w tamtym commicie.

[Research i decyzja o komunikacji](docs/agent-communication-research.md) opisują źródła,
alternatywy, ograniczenia i zakres weryfikacji. Nowe skille dodawaj jako samodzielne
`skills/<name>/SKILL.md` z własnymi zasobami; nie odwołuj się do plików poza katalogiem
skilla, ponieważ installer może pobrać tylko ten jeden katalog.

# Komunikacja agentów: research i decyzja

Stan sprawdzenia: 2026-11-10. Zakres: osobne interaktywne sesje CLI, wspólny lokalny
projekt, driver i niezależni recenzenci. Przegląd dokumentacji i kodu upstream nie jest
benchmarkiem jakości modeli ani dowodem działania każdej konfiguracji.

**Decyzja (2026-11-10): pierwsza wersja nie budzi sesji.** Wspólna skrzynka zostaje,
a watcher, polecenie `bind` i adaptery `codex queue` / `opencode api` zostały usunięte.
Sesję, która ma coś do przeczytania, wznawia użytkownik nadzorujący run. Powód: budzenia
nie udało się sprawdzić na prawdziwych sesjach, zależało od usług sesji konkretnego
dostawcy i wymagało ręcznego podawania identyfikatorów sesji. Usunięty kod jest
w `mrstroz/claude-code-plugins`, gałąź `feat/team-plugin`, commit `55a0e27`.

## Co wynika ze źródeł

| Rozwiązanie / źródło | Mechanizm | Wniosek dla tego repozytorium |
|---|---|---|
| [Claude Code Agent Teams](https://code.claude.com/docs/en/agent-teams) | Własne sesje, lista zadań i skrzynki JSON; walidacja wiadomości i obsługa błędów zapisu | Dobry mechanizm wewnątrz Claude. Nie należy opierać wspólnego protokołu na ręcznych zmianach jego stanu wewnętrznego. |
| [Codex developer commands](https://learn.chatgpt.com/docs/developer-commands) oraz lokalne `codex queue --help` | Wiadomość kierowana do dokładnej sesji; opcjonalne wznowienie | Nie używamy w pierwszej wersji. Lokalna pomoc ostrzega, że nieudane wznowienie może nadal dać kod 0, więc sukces polecenia nie dowodzi obudzenia sesji. |
| [Codex App Server](https://learn.chatgpt.com/docs/app-server) | Wątki, tury, start/resume, steer i strumień zdarzeń | Przydatny dla własnego supervisora zarządzającego sesjami. Większy zakres niż skill do istniejących CLI. Nie uruchamiać drugiego procesu modelu na tej samej rozmowie. |
| [OpenCode V2 API](https://opencode.ai/docs/api), [schemat](https://opencode.ai/docs/openapi.json), [kontrakt sesji w kodzie](https://github.com/anomalyco/opencode/blob/dev/specs/v2/session.md) | Trwałe przyjęcie wejścia przed wykonaniem; `queue` lub `steer`; `resume` steruje uruchomieniem | Nie używamy w pierwszej wersji. Ewentualny adapter: `session.prompt` z `delivery: queue`, `resume: true`; API oznaczone jako eksperymentalne, nie mieszać z V1. |
| [agent-talk](https://github.com/xhluca/agent-talk), [adapter OpenCode](https://github.com/xhluca/agent-talk/blob/main/extensions/opencode/inbox-monitor.ts), [odbiór Codex](https://github.com/xhluca/agent-talk/blob/main/docs/codex-auto-receive.md) | Relay/spool oraz osobne mechanizmy integracji z hostami | Potwierdza rozdzielenie transportu, trwałego odbioru i budzenia. Hook działa, gdy host go wywoła; sam nie gwarantuje budzenia bezczynnej sesji. Część adapterów zależy od konkretnej generacji hosta. |
| [MCP Agent Mail Rust](https://github.com/Dicklesworthstone/mcp_agent_mail_rust) | MCP, trwały magazyn, wyszukiwalne wątki, tożsamości i rezerwacje plików | Warto rozważyć przy wielu projektach i rozbudowanych zespołach. Dla jednego lokalnego repo oznacza dodatkowy serwer i instalację. Starszy projekt Python oznaczono jako zastąpiony wersją Rust. |
| [agentdrop](https://github.com/Interwork-ai/agentdrop) | Wspólna skrzynka dla ludzi i agentów różnych narzędzi | Pasuje do przekazywania wyników między osobami; szerszy produkt niż lokalne bramki rewizji. |
| [hcom](https://github.com/aannoo/hcom) | Hooki hosta; automatyczne dostarczanie do Claude Code, Codex, Gemini CLI i OpenCode, także do bezczynnych sesji; potrafi uruchamiać agentów | Najdojrzalszy znaleziony transport między różnymi CLI. Wymaga instalacji binarki i globalnych hooków. Pierwszy kandydat, jeśli ręczne wznawianie okaże się uciążliwe. Mechanizm znany z dokumentacji, nie z własnej próby. |
| [agmsg](https://github.com/fujibee/agmsg) | bash + sqlite, instalowany jako Agent Skill | Najbliższy filozofią (lokalnie, bez serwera). Sposobu budzenia bezczynnej sesji nie zweryfikowano. |
| [Thrum](https://github.com/leonletto/thrum) | Wiadomości w Git, między sesjami, worktree i maszynami | Do rozważenia, gdy sesje przestaną dzielić jeden lokalny katalog. |
| [Agent Client Protocol](https://agentclientprotocol.com/protocol/v1/prompt-turn) | Klient inicjuje prompt w utworzonej sesji, agent strumieniuje zdarzenia | Możliwa podstawa przyszłego launchera. Nie daje sam z siebie wspólnej skrzynki dowolnych otwartych terminali. |

Lokalnie odczytano pomoc Codexa 0.167.0, Claude Code 2.1.322 i OpenCode 2.2.5.
Z OpenCode pobrano publiczny schemat żądania `POST /api/session/{sessionID}/prompt`:
obowiązkowe `text`, opcjonalne `id`, `delivery`, `resume` i inne załączniki. `opencode api`
obsługuje identyfikator operacji, `--param`, `--data` i `--server`. To kontrakt
na wypadek powrotu do adaptera; obecna wersja z niego nie korzysta.

## Alternatywy

1. **Natywne zespoły jednego dostawcy.** Najmniejsza integracja, gdy wszystkie sesje
   należą do jednego hosta. Nie rozwiązują naszego wymagania współpracy różnych CLI.
2. **MCP / relay / osobna usługa.** Ułatwia komunikację między komputerami i zarządzanie
   tożsamościami. Wprowadza instalację serwera, uprawnienia i zależności operacyjne.
   Samo udostępnienie narzędzia odczytu skrzynki nadal nie dowodzi budzenia hosta.
3. **Supervisor uruchamiający wszystkie modele przez SDK / ACP.** Kontroluje cykl życia,
   timeouty i dostarczanie wejść. To sensowny kierunek dla autonomicznego launchera,
   lecz zmienia sposób uruchamiania CLI i zakres projektu.
4. **Wspólna lokalna skrzynka, wznawianie przez użytkownika.** Zachowuje istniejące
   interaktywne sesje i nie zależy od żadnego dostawcy. Ten wariant wybieramy. Adaptery
   budzenia (własne albo hooki w stylu hcom) można dołożyć później bez zmiany protokołu,
   bo treść wiadomości zawsze zostaje w dzienniku.
5. **tmux send-keys / prywatne pliki sesji.** Wprowadzają zależność od terminala, fokusu,
   formatu wewnętrznego i aktualnego stanu UI. Nie są podstawą tego rozwiązania.

To decyzja dla tego zakresu, nie twierdzenie o jednym najlepszym mechanizmie dla
wszystkich zespołów. Jeżeli wymaganiem stanie się automatyczne uruchamianie agentów na
różnych komputerach, trzeba ponownie ocenić supervisor lub serwer skrzynki.

## Inwarianty i zmiany względem pluginu

- `events.jsonl` pozostaje trwałym źródłem prawdy, z serializacją zapisów, naprawą
  niedokończonego ogona oraz regenerowanymi widokami. Snapshoty i werdykty mają rewizje.
- Odbiór ma semantykę **co najmniej raz**. `wait` może zwrócić tę samą partię po awarii.
  `--seen` potwierdza odczytaną partię; identyfikatory zdarzeń i `--op` pozwalają obsłużyć
  powtórzenia. Nie deklarujemy exactly-once dla działań zewnętrznych agenta.
- Nic nie budzi bezczynnej sesji. Claude Code trzyma `wait` jako zadanie w tle, którego
  zakończenie rozpoczyna turę; pozostałe sesje czytają po wznowieniu przez użytkownika.
- `post` wypisuje `not waiting: <id>` dla adresatów bez żywego procesu `wait`. Znacznik
  po zabitym `wait` nie liczy się, bo sprawdzany jest proces, który go zapisał.
- Wiadomość od innego agenta jest danymi zadania, nie poleceniem użytkownika ani nowym
  uprawnieniem.
- Umowa komunikacji pozostaje lokalna i kooperatywna. Dostęp do katalogu runu umożliwia
  zmianę jego danych; to nie izolacja od złośliwego procesu mającego te same uprawnienia.
- Instrukcje ścieżek są niezależne od `${CLAUDE_PLUGIN_ROOT}` i `$ARGUMENTS`. Cały skill
  mieści się w jednym katalogu instalowanym przez `npx skills`.

## Weryfikacja i granice

Testy procesowe obejmują trwałość i współbieżność protokołu, rewizje, potwierdzenia,
przerwane `wait` oraz informację o adresatach, którzy nie czekają. Nie rozmawiają
z modelami.

Nie wykonano jeszcze prawdziwego runu z kilkoma sesjami różnych CLI ani sprawdzenia, czy
OpenCode widzi skill zainstalowany w `~/.agents/skills/`. To pierwsza rzecz do zrobienia
przed dalszym rozwojem.

Wynik lokalny: 51/51 testów procesowych przeszło; kontrola składni Node przeszła;
`skills@1.7.6 add . --list` wykrywa `team`. Walidacja podstawowego frontmatter przeszła
po pominięciu w kopii walidacyjnej rozszerzenia Claude `disable-model-invocation`,
którego walidator Codexa nie dopuszcza w swojej liście kluczy. Oryginalny skill zachowuje
to rozszerzenie oraz politykę jawnego wywołania w `agents/openai.yaml`.
Instalacja globalna przez `skills@1.7.6` w odizolowanym katalogu domowym przeszła
(kopia w `~/.agents/skills/team`, dowiązanie w `~/.claude/skills/team`); odinstalowania
nie sprawdzano.

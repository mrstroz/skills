# Komunikacja agentów: research i decyzja

Stan sprawdzenia: 2026-11-10. Zakres: osobne interaktywne sesje CLI, wspólny lokalny
projekt, driver i niezależni recenzenci. Przegląd dokumentacji i kodu upstream nie jest
benchmarkiem jakości modeli ani dowodem działania każdej konfiguracji.

## Co wynika ze źródeł

| Rozwiązanie / źródło | Mechanizm | Wniosek dla tego repozytorium |
|---|---|---|
| [Claude Code Agent Teams](https://code.claude.com/docs/en/agent-teams) | Własne sesje, lista zadań i skrzynki JSON; walidacja wiadomości i obsługa błędów zapisu | Dobry mechanizm wewnątrz Claude. Nie należy opierać wspólnego protokołu na ręcznych zmianach jego stanu wewnętrznego. |
| [Codex developer commands](https://learn.chatgpt.com/docs/developer-commands) oraz lokalne `codex queue --help` | Wiadomość kierowana do dokładnej sesji; opcjonalne wznowienie | Używać publicznego polecenia, ale wymagać potwierdzenia z naszej skrzynki. Lokalna pomoc ostrzega, że nieudane wznowienie może nadal dać kod 0. |
| [Codex App Server](https://learn.chatgpt.com/docs/app-server) | Wątki, tury, start/resume, steer i strumień zdarzeń | Przydatny dla własnego supervisora zarządzającego sesjami. Większy zakres niż skill do istniejących CLI. Nie uruchamiać drugiego procesu modelu na tej samej rozmowie. |
| [OpenCode V2 API](https://opencode.ai/docs/api), [schemat](https://opencode.ai/docs/openapi.json), [kontrakt sesji w kodzie](https://github.com/anomalyco/opencode/blob/dev/specs/v2/session.md) | Trwałe przyjęcie wejścia przed wykonaniem; `queue` lub `steer`; `resume` steruje uruchomieniem | Wysyłać krótki sygnał przez `session.prompt` z `delivery: queue`, `resume: true`; nie mieszać z API V1. |
| [agent-talk](https://github.com/xhluca/agent-talk), [adapter OpenCode](https://github.com/xhluca/agent-talk/blob/main/extensions/opencode/inbox-monitor.ts), [odbiór Codex](https://github.com/xhluca/agent-talk/blob/main/docs/codex-auto-receive.md) | Relay/spool oraz osobne mechanizmy integracji z hostami | Potwierdza rozdzielenie transportu, trwałego odbioru i budzenia. Hook działa, gdy host go wywoła; sam nie gwarantuje budzenia bezczynnej sesji. Część adapterów zależy od konkretnej generacji hosta. |
| [MCP Agent Mail Rust](https://github.com/Dicklesworthstone/mcp_agent_mail_rust) | MCP, trwały magazyn, wyszukiwalne wątki, tożsamości i rezerwacje plików | Warto rozważyć przy wielu projektach i rozbudowanych zespołach. Dla jednego lokalnego repo oznacza dodatkowy serwer i instalację. Starszy projekt Python oznaczono jako zastąpiony wersją Rust. |
| [agentdrop](https://github.com/Interwork-ai/agentdrop) | Wspólna skrzynka dla ludzi i agentów różnych narzędzi | Pasuje do przekazywania wyników między osobami; szerszy produkt niż lokalne bramki rewizji. |
| [Agent Client Protocol](https://agentclientprotocol.com/protocol/v1/prompt-turn) | Klient inicjuje prompt w utworzonej sesji, agent strumieniuje zdarzenia | Możliwa podstawa przyszłego launchera. Nie daje sam z siebie wspólnej skrzynki dowolnych otwartych terminali. |

Lokalnie odczytano pomoc Codexa 0.167.0, Claude Code 2.1.322 i OpenCode 2.2.5.
Z OpenCode pobrano publiczny schemat żądania `POST /api/session/{sessionID}/prompt`:
obowiązkowe `text`, opcjonalne `id`, `delivery`, `resume` i inne załączniki. `opencode api`
obsługuje identyfikator operacji, `--param`, `--data` i `--server`. To dokładny kontrakt
adaptera; nie korzystamy z nieudokumentowanego dopisywania do bazy sesji.

## Alternatywy

1. **Natywne zespoły jednego dostawcy.** Najmniejsza integracja, gdy wszystkie sesje
   należą do jednego hosta. Nie rozwiązują naszego wymagania współpracy różnych CLI.
2. **MCP / relay / osobna usługa.** Ułatwia komunikację między komputerami i zarządzanie
   tożsamościami. Wprowadza instalację serwera, uprawnienia i zależności operacyjne.
   Samo udostępnienie narzędzia odczytu skrzynki nadal nie dowodzi budzenia hosta.
3. **Supervisor uruchamiający wszystkie modele przez SDK / ACP.** Kontroluje cykl życia,
   timeouty i dostarczanie wejść. To sensowny kierunek dla autonomicznego launchera,
   lecz zmienia sposób uruchamiania CLI i zakres projektu.
4. **Wspólna lokalna skrzynka + adaptery.** Zachowuje istniejące interaktywne sesje,
   umożliwia stopniową integrację i odzyskanie stanu. Ten wariant wybieramy.
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
- Budzenie jest wskazówką do odczytu dziennika. Nie przenosi treści od innego agenta
  jako promptu użytkownika. Sukces adaptera nie potwierdza wykonania pracy.
- Każda rola, także driver, może jawnie powiązać sesję przez `bind`. Samo odziedziczenie
  zmiennej Codexa nie aktywuje adaptera w innym CLI.
- Watcher ponawia niepotwierdzone wiadomości, także po przerwaniu tuż po ich wypisaniu.
  Zmiana powiązania sesji resetuje stan ponowień. Retry ma limit, awarie są widoczne.
- Wywołania adapterów są asynchroniczne i ograniczone czasowo; heartbeat watchera
  działa podczas oczekiwania na CLI. Polling plików wykonuje proces Node, nie model.
- Umowa komunikacji pozostaje lokalna i kooperatywna. Dostęp do katalogu runu umożliwia
  zmianę jego danych; to nie izolacja od złośliwego procesu mającego te same uprawnienia.
- Instrukcje ścieżek są niezależne od `${CLAUDE_PLUGIN_ROOT}` i `$ARGUMENTS`. Cały skill
  mieści się w jednym katalogu instalowanym przez `npx skills`.

## Weryfikacja i granice

Testy procesowe obejmują trwałość i współbieżność protokołu, rewizje, potwierdzenia,
restart watchera, ograniczone ponowienia i kontrakty wywołania adapterów. Symulowany
adapter potwierdza argumenty i zachowanie po błędzie; nie zastępuje rzeczywistej rozmowy
z modelem. Wersję instalatora i wyniki końcowe podaje README / raport wykonania.

Do uruchomienia zespołu wymagamy próby komunikacji w obie strony z odbiorcą będącym
bezczynnym, włącznie z driverem. Bez niej konfiguracja jest przygotowana, lecz nie
potwierdzona operacyjnie. Nie wykonano płatnego testu trzech rzeczywistych sesji modeli.
Na CLI bez obsługi powiadomień pozostaje jawny odbiór. OpenCode V2 oznacza API jako
eksperymentalne, dlatego po aktualizacji trzeba powtórzyć próbę kontraktu i komunikacji.

Wynik lokalny: 62/62 testy procesowe przeszły; kontrola składni Node przeszła;
`skills@1.7.6 add . --list` wykrywa `team`. Walidacja podstawowego frontmatter przeszła
po pominięciu w kopii walidacyjnej rozszerzenia Claude `disable-model-invocation`,
którego walidator Codexa nie dopuszcza w swojej liście kluczy. Oryginalny skill zachowuje
to rozszerzenie oraz politykę jawnego wywołania w `agents/openai.yaml`.
Pełna instalacja/odinstalowanie globalne oraz próba z trzema rzeczywistymi sesjami
pozostają do wykonania. Na prośbę użytkownika pracę zakończono na tym etapie.

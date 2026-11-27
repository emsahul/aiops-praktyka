# Incydent: wolne menu i błędy w Kantynie

- **Namespace:** `ewa`
- **Zgłoszenie:** użytkownicy piszą, że menu ładuje się bardzo wolno, a czasem dostają błąd.
- **Start incydentu (wykrycie):** 2026-11-27 15:27 CET
- **Pierwsze błędy w logach orders-api:** 15:24:52 CET (`Exception in ASGI application`)

## Objawy (krok 1)

`setup/check.sh` → **8/11**:

- ❌ Deployment `kantyna-orders-api`: gotowe 0/2 (pody `Running`, ale nie `Ready`)
- ❌ Aplikacja: menu przez web, błąd
- ❌ Aplikacja: zamówienie przez web, błąd
- ✅ web, payments, worker, postgres, Service'y z endpointami

Dodatkowo:

- eventy: `Readiness probe failed: Get ".../readyz": context deadline exceeded` na obu podach orders-api
- logi orders-api: co 5 s `Readiness check failed`, `"error": "couldn't get a connection after 30.00 sec"`
- `kantyna-loadgen` uruchomiony ok. 15:22 (5 min przed wykryciem)

## Dziennik

| Godzina | Hipoteza (AI / ja) | Komenda / zapytanie | Wynik | Status |
|---|---|---|---|---|
| 15:27 | (wykrycie) | `setup/check.sh` | 8/11: orders-api 0/2 gotowe, menu i zamówienie przez web nie działają | objaw |
| 15:27 | (wykrycie) | `kubectl get events --sort-by=.lastTimestamp` | readiness probe orders-api: timeout na `/readyz` | objaw |
| 15:28 | (wykrycie) | `kubectl logs deploy/kantyna-orders-api --since=3h` | pierwszy błąd 15:24:52; potem `couldn't get a connection after 30.00 sec` | objaw, przyczyna niepotwierdzona |
| 15:29 | (komunikat) | `docs/prompts/komunikat-incydentu.md` | pierwszy komunikat na kanał zespołu, niżej | wysłany |
| 15:35 | H1 (AI): wyczerpana pula połączeń DB w orders-api | traceback z `kubectl logs kantyna-orders-api-56f4c6769c-wxt9t --since=3h` | `get_menu` → `db.fetch_menu` → `psycopg_pool.PoolTimeout: couldn't get a connection after 30.00 sec` | objaw potwierdzony |
| 15:35 | H1 (AI) | `kubectl logs <pod> --since=3h \| grep -c 'pool\|couldn.t get a connection'` | 96 (`65tmq`) + 83 (`wxt9t`), więc oba pody, a nie jeden | potwierdzona |
| 15:36 | H1/H2 (AI, runbook krok 4 „Recent changes”) | `kubectl get cm kantyna-flags -o jsonpath='{.data.flags\.json}'` | orders-api: `db_pool_leak: true`, `slow_menu_query: true` | **przyczyna potwierdzona** |
| 15:36 | H1/H2 (ja): skąd zmiana? | `kubectl get cm kantyna-flags -o jsonpath='{.metadata.managedFields[*].manager} {.metadata.managedFields[*].time}'` | ostatni zapis: `kubectl-patch` o 14:22:52Z = **15:22:52 CET**, 2 min przed pierwszym błędem | potwierdzona (ręczny patch poza pipeline) |
| 15:37 | (ja): czy to było wdrożenie? | `helm history kantyna` | ostatni release rev 2, 26.11 09:05, w czasie incydentu nie było wdrożenia | wykluczone |
| 15:37 | (ja): co mówi repo? | `app/deploy/helm/kantyna/values.yaml` | `db_pool_leak: false`, `slow_menu_query: false`, więc klaster rozjechał się z repo | dryf potwierdzony |
| 15:38 | H1 (ja): mechanizm | kod `app/orders-api/app/main.py:147-151`, `db.py:214` | przy `db_pool_leak` 30% żądań `/api/menu` pobiera połączenie i go nie oddaje; pula `DB_POOL_MAX=10` (`config.py:18`) | potwierdzona |
| 15:38 | H2 (ja): mechanizm | kod `app/orders-api/app/runtime.py:52`, `db.py:116-128` | przy `slow_menu_query` menu robi N+1 zapytań (SELECT na każdą pozycję), więc dłużej trzyma połączenie | potwierdzona (flaga + kod); wpływ na p95 do sprawdzenia w Grafanie |
| 15:39 | H3 (AI): awaria Postgresa / sieci | `kubectl get pods -o wide`, `setup/check.sh` | `kantyna-postgres-0` Running 1/1, 0 restartów, Service ma endpointy, payments 2/2; błąd to timeout puli klienta, a nie odmowa połączenia | odrzucona |
| 15:39 | H4 (ja): sam ruch z loadgen | `kubectl get events` | loadgen wystartował ok. 15:22 i generuje wywołania `/api/menu`, które uruchamiają wyciek; bez flagi pula 10 wystarcza | odrzucona jako przyczyna, **czynnik wyzwalający** |
| 15:41:24 | Mitygacja 1/2: wyłączenie obu flag (wartości jak w repo) | `kubectl patch cm kantyna-flags --type merge --patch-file flags-patch.json` (`db_pool_leak`, `slow_menu_query` → `false`) | `configmap/kantyna-flags patched` | wykonane |
| 15:41:32 | Mitygacja 2/2: zwolnienie wyciekłych połączeń | `kubectl rollout restart deploy/kantyna-orders-api` + `rollout status` | `successfully rolled out` | wykonane |
| 15:42:47 | Weryfikacja mitygacji | `setup/check.sh` | **SUKCES: 11/11** | usługa przywrócona |
| 15:45 | Naprawa (ja): pipeline z lab03 nie był zacommitowany, a `--reuse-values` nie wymuszało flag z repo | `helm template ... --set-json "flags=<.flags z values.yaml>"` + `helm lint` | ConfigMap z flagami `false`; `0 chart(s) failed` | commit niżej |
| 15:55 | Naprawa, commit `f7087ec` | run [46019927747](https://github.com/emsahul/aiops-praktyka/actions/runs/46019927747) | ❌ `helm upgrade` rev 3 `failed`: konflikt server-side apply z `kubectl-patch` (`kantyna-flags .data.flags.json`) i `kubectl-edit` (`kantyna-web` image, ręczna zmiana z 25.11). Częściowo wdrożone: orders-api/payments/worker na `:f7087ec`, `check.sh` dalej 11/11 | nieudany |
| 16:00 | (ja): Helm 4 nie nadpisuje ręcznych zmian bez `--force-conflicts` | `helm history kantyna` | opis rev 3: `Apply failed with 1 conflict: conflict with "kubectl-patch"` | potwierdzona; dodaję `--force-conflicts` |
| 16:03 | **Naprawa, commit `5742bbb`** (`5742bbb9a6952e84fa2674c73bd46315b71f89f2`) | run [46021241476](https://github.com/emsahul/aiops-praktyka/actions/runs/46021241476) | ✅ wszystkie joby `success`; helm rev 4 `deployed`; 4 serwisy na `:5742bbb…`; właścicielem `kantyna-flags` jest tylko `helm Apply` (zniknął `kubectl-patch`); flagi `false` | **naprawione** |
| 16:04:29 | Weryfikacja po naprawie | `setup/check.sh` | **SUKCES: 11/11** | OK; trzeba jeszcze 5 min stabilnych metryk w Grafanie (krok 5) |

## Naprawa (krok 4)

- **Mitygacja:** 15:41 ręczne wyłączenie flag + restart orders-api, usługa wróciła o 15:42:47.
- **Naprawa (commity w forku):**
  - `f7087ec`: pipeline z lab03 dodany do repo, `helm upgrade` dostaje `--set-json flags=<.flags z values.yaml>`. Run [46019927747](https://github.com/emsahul/aiops-praktyka/actions/runs/46019927747) ❌ przez konflikty SSA.
  - `5742bbb`: `--force-conflicts`. Run [46021241476](https://github.com/emsahul/aiops-praktyka/actions/runs/46021241476) ✅.
- **Skutek uboczny do post-mortem:** wdrożenie nadpisało też ręczną zmianę obrazu `kantyna-web` (`kubectl-edit` z 25.11, `web:1.0.0`). Teraz web działa na obrazie z pipeline'u.

## Pierwszy komunikat (krok 2)

### Wersja robocza od AI

> Od ok. 15:25 Kantyna ma problemy z bazą danych: wyczerpała się pula połączeń orders-api do Postgresa. Menu ładuje się wolno albo wcale, zamówienia nie przechodzą. Wszyscy użytkownicy są dotknięci. Restartujemy orders-api i zwiększamy pulę połączeń, poprawka będzie w ciągu 30 minut. Następna aktualizacja o 16:00.

### Poprawki i powody

- **Usunięta przyczyna** („wyczerpana pula połączeń”). Na razie to tylko hipoteza z jednego komunikatu w logach, nie mamy jej potwierdzonej.
- **Usunięte działania i termin naprawy** („restartujemy, zwiększamy pulę, poprawka w 30 min”). Nic takiego jeszcze nie zostało postanowione ani sprawdzone.
- **„Wszyscy użytkownicy”** zamienione na to, co zmierzyliśmy: oba pody orders-api są niegotowe, a sprawdzenie menu i zamówienia przez web kończy się błędem.
- **Bez nazw wewnętrznych** (Postgres, pule, nazwy podów). Odbiorca to zespół, a komunikat ma być czytelny bez znajomości klastra.
- **Termin aktualizacji** to konkretna godzina, a nie „wkrótce”. Obiecujemy aktualizację, a nie naprawę.

### Wersja finalna (na kanał zespołu)

> **[Incydent] Kantyna: menu i zamówienia nie działają.** Od ok. 15:25 menu Kantyny ładuje się bardzo wolno albo kończy się błędem, a zamówienia przez stronę nie przechodzą. Wpływ: nasze testy o 15:27 nie mogły ani wczytać menu, ani złożyć zamówienia, więc zakładamy, że dotyczy to wszystkich użytkowników. Przyczyny jeszcze nie znamy i jej nie potwierdziliśmy; trwa diagnoza po stronie serwisu obsługującego menu i zamówienia. Następna aktualizacja o 16:00 albo wcześniej, jeśli sytuacja się zmieni.

## Diagnoza (krok 3)

Runbooki: `app/runbooks/KantynaDBPoolExhausted.md` (główny: timeout puli, padające `/readyz`, pody wypadają z endpointów) i `app/runbooks/KantynaHighLatencyP95.md` (pomocniczy: wolne menu, wolne zapytania DB). W obu runbookach krok „Recent changes” każe sprawdzić `kantyna-flags`.

### Prompt (z `docs/prompts/diagnoza.md`)

```text
Jesteś SRE. Środowisko: AWS EKS (eu-central-1), namespace ewa, komponent orders-api 1.0.0 (FastAPI, psycopg_pool, DB_POOL_MAX=10).
Objaw: od 15:24:52 CET menu ładuje się bardzo wolno albo zwraca błąd, zamówienia przez web nie przechodzą.
Oba pody orders-api 0/1 Ready. Ostatnio: ok. 15:22 uruchomiony loadgen (ruch syntetyczny).
Dane:
- events: Readiness probe failed: Get ".../readyz": context deadline exceeded (oba pody)
- logs: get_menu -> db.fetch_menu -> psycopg_pool.PoolTimeout: couldn't get a connection after 30.00 sec
- logs: co 5 s "Readiness check failed", error "couldn't get a connection after 30.00 sec"
Już sprawdziłem: web, payments, worker, postgres Ready; Service'y mają endpointy (setup/check.sh 8/11).
Zadanie: podaj 3 najbardziej prawdopodobne hipotezy, od najbardziej prawdopodobnej.
Ograniczenia: proponuj tylko komendy read-only. Nie proponuj jeszcze poprawki.
Jeśli brakuje danych, napisz, jakiej komendy potrzebujesz, zamiast zgadywać.
Format: tabela | hipoteza | dowód z danych powyżej | komenda weryfikująca |.
```

### Hipotezy i wynik weryfikacji

| # | Hipoteza | Źródło | Dowód | Komenda weryfikująca | Status |
|---|---|---|---|---|---|
| H1 | Połączenia wyciekają z puli orders-api (flaga `db_pool_leak`) | AI + runbook DBPoolExhausted | `PoolTimeout` na obu podach; flaga `true` w ConfigMap; kod nie oddaje połączenia (`db.py:214`) | `kubectl get cm kantyna-flags -o jsonpath='{.data.flags\.json}'` | **przyczyna potwierdzona** |
| H2 | Wolne zapytanie menu N+1 (flaga `slow_menu_query`) | AI + runbook HighLatencyP95 | flaga `true` w ConfigMap; N+1 w `db.fetch_menu` (`db.py:116-128`) | jw. + p95 `/api/menu` w Grafanie | **przyczyna potwierdzona** (wpływ na p95 do sprawdzenia na wykresie) |
| H3 | Postgres lub sieć do bazy nie działa | AI | postgres Ready, 0 restartów, endpointy OK; błąd to timeout po stronie klienta | `kubectl get pods`, `setup/check.sh` | odrzucona |
| H4 | Sam wzrost ruchu (loadgen) | ja | loadgen od ok. 15:22; to on wywołuje `/api/menu` i uruchamia wyciek | `kubectl get events` | czynnik wyzwalający, nie przyczyna |

### Wnioski

- **Dwie przyczyny:** `db_pool_leak: true` (wyczerpanie puli, które powoduje błędy i niegotowe pody) oraz `slow_menu_query: true` (wolne menu).
- **Źródło:** ręczny `kubectl patch` ConfigMap `kantyna-flags` o 15:22:52 CET, poza repo i pipeline'em. W repo obie flagi mają wartość `false`, więc konfiguracja się rozjechała (do post-mortem).
- **Do weryfikacji przez człowieka (agent nie ma dostępu):**
  - Grafana: `max by (pod)(db_pool_connections_in_use{namespace="ewa"}) / max by (pod)(db_pool_size{namespace="ewa"})`. Oczekiwane: wzrost do 1.0 od ok. 15:23 bez powrotu. Uwaga: runbook ma `namespace="kantyna"`, trzeba podmienić na `ewa`.
  - `pg_stat_activity` z runbooka wymaga `kubectl exec`, który jest zablokowany w `.claude/settings.json`. Można to uruchomić samemu.
- **Kierunek dla kroku 4 (jeszcze nie wykonany):** przywrócić flagi z repo przez pipeline, a potem zrestartować orders-api, bo wyciekłe połączenia (`_held_connections`) nie wrócą same po wyłączeniu flagi. Nie skalować orders-api w górę (ostrzeżenie z runbooka: każda replika otwiera własną pulę).

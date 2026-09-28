# MemPalace MCP — Docker (Windows / Docker Desktop)

Serwer MCP [MemPalace](https://pypi.org/project/mempalace/) (`mempalace-mcp`) uruchamiany
w kontenerze Docker. Udostępnia pamięć/wiedzę projektów przez protokół MCP (transport HTTP,
endpoint `/mcp`) i jest gotowy do podpięcia pod Claude Desktop na tym samym hoście.

---

## 1. Wymagania

- **Docker Desktop** dla Windows z backendem **WSL2** (Settings → General → *Use the WSL 2 based engine*).
- Do metody HTTP z sekcji 6: Node.js (dla `npx`). Metoda `docker exec` z sekcji 3 go nie wymaga.

## 2. Uruchomienie

1. Otwórz PowerShell w katalogu projektu.

2. Utwórz plik `.env` z własnym, losowym tokenem:

   ```powershell
   Copy-Item .env.example .env
   -join ((48..57)+(65..90)+(97..122) | Get-Random -Count 40 | % {[char]$_})
   ```

   Wygenerowany ciąg wpisz w `.env` jako wartość `MEMPALACE_MCP_HTTP_TOKEN`.

3. **Jednorazowo** (raz na cały Docker Desktop) utwórz zewnętrzną sieć:

   ```powershell
   docker network create shared-mcp-net
   ```

   Jeśli sieć już istnieje, polecenie zwróci błąd `already exists` — można go zignorować.

4. Zbuduj obraz i uruchom kontener w tle:

   ```powershell
   docker compose up -d --build
   ```

5. Sprawdź, czy kontener działa:

   ```powershell
   docker compose ps
   docker compose logs -f
   curl http://localhost:8002/healthz
   ```

   Healthcheck po chwili powinien pokazać status `healthy`.

Dane (indeks pamięci, `chroma.sqlite3`, `knowledge_graph.sqlite3`, `mempalace.yaml`)
leżą w `./palace-data` i przetrwają restart oraz przebudowę kontenera.

### Zatrzymanie / restart

```powershell
docker compose down           # zatrzymuje i usuwa kontener (dane w palace-data zostają)
docker compose restart        # restart bez przebudowy
docker compose up -d --build  # przebudowa po zmianie Dockerfile
```

---

## 3. Podłączenie do Claude Desktop (`docker exec`, stdio)

Plik konfiguracyjny Claude Desktop na Windows:

```
%APPDATA%\Claude\claude_desktop_config.json
```

Dodaj serwer:

```json
{
  "mcpServers": {
    "mempalace": {
      "command": "docker",
      "args": [
        "exec",
        "-i",
        "mempalace-mcp",
        "python",
        "-m",
        "mempalace.mcp_server",
        "--palace",
        "/palace"
      ]
    }
  }
}
```

Kontener `mempalace-mcp` musi działać (`docker compose up -d`), zanim Claude Desktop spróbuje
się połączyć. Po zapisaniu configu zamknij Claude Desktop całkowicie (z paska zadań) i uruchom
ponownie.

**Test:** poproś Claude o sprawdzenie statusu palace — jeśli zwróci listę wings/rooms,
połączenie działa.

---

## 4. Struktura projektu

```
mem-palace/
├── Dockerfile           # obraz: python:3.12-slim + pakiet pip "mempalace"
├── docker-compose.yml   # usługa mempalace-mcp, sieć shared-mcp-net, healthcheck
├── .env                 # MEMPALACE_MCP_HTTP_TOKEN (lokalny; wzór: .env.example)
├── .env.example         # szablon .env
├── .dockerignore        # wyklucza palace-data/ i .env z kontekstu builda
├── .gitignore
├── LICENSE              # MIT
├── README.md
└── palace-data/         # wolumen z danymi (montowany do /palace w kontenerze)
```

`palace-data/` jest jedynym trwałym stanem — obraz i kontener można w każdej chwili odtworzyć
przez `docker compose up -d --build`.

---

## 5. Kluczowe ustawienia

- **`mempalace==3.5.0`** — pierwsza wersja pakietu z opcjonalnym transportem HTTP
  (`--transport http`).
- **Bez `mempalace init` w Dockerfile** — `mempalace-mcp` sam tworzy strukturę palace
  (`mempalace.yaml`, `chroma.sqlite3`) przy pierwszym starcie, jeśli `/palace` jest puste.
- **Zakomentowane `user: "${UID:-1000}:${GID:-1000}"`** w `docker-compose.yml` — na
  Windows/Docker Desktop (WSL2) bind mount nie daje użytkownikowi UID 1000 uprawnień do
  odczytu `/palace` (`PermissionError`). Kontener działa jako root. Na Linuksie można tę linię
  odkomentować, żeby pliki w `palace-data` miały właściwego właściciela.
- **Port `8002` → `8000`** — mapowanie portu hosta na port kontenera; inne kontenery w sieci
  łączą się po nazwie: `http://mempalace-mcp:8000/mcp`.
- **Sieć `shared-mcp-net`** jest zewnętrzna (`external: true`), więc Compose jej nie tworzy —
  patrz krok 3 w sekcji 2.

---

## 6. Alternatywa: dostęp przez HTTP

Endpoint `http://localhost:8002/mcp` jest chroniony tokenem z `.env` (nagłówek
`Authorization: Bearer <token>`). Mogą z niego korzystać klienci MCP obsługujący transport
HTTP, np. inne kontenery w sieci `shared-mcp-net`.

---

## 7. Rozwiązywanie problemów

| Objaw | Rozwiązanie |
|---|---|
| `network shared-mcp-net not found` przy `docker compose up` | Utwórz sieć: `docker network create shared-mcp-net`. |
| `PermissionError` przy odczycie plików w `/palace` | Zostaw linię `user:` w `docker-compose.yml` zakomentowaną (Windows). |
| Port `8002` zajęty | Zmień mapowanie portu w `docker-compose.yml`, np. `"8010:8000"`. |
| Healthcheck nie przechodzi w `healthy` | Sprawdź `docker compose logs -f` i czy obraz zbudował się bez błędów. |
| Claude Desktop nie widzi serwera `mempalace` | Sprawdź poprawność JSON w `claude_desktop_config.json`, czy kontener działa (`docker compose ps`), i zrestartuj Claude Desktop całkowicie (wyjście z paska zadań). |
| Docker Desktop nie startuje / błędy WSL2 | Włącz *Use the WSL 2 based engine* i zaktualizuj WSL: `wsl --update`. |

---

## 8. Bezpieczeństwo

- **Zawsze ustaw token** `MEMPALACE_MCP_HTTP_TOKEN` w `.env`. Pusty token oznacza brak
  autoryzacji: endpoint `/mcp` jest wtedy dostępny bez uwierzytelnienia dla każdego kontenera
  w sieci `shared-mcp-net` i każdego procesu na hoście przez port `8002`.
- Jeśli port `8002` nie jest potrzebny, usuń sekcję `ports:` z `docker-compose.yml` i korzystaj
  z serwera wyłącznie przez sieć Docker lub `docker exec`.

## Licencja

MIT — patrz [LICENSE](LICENSE).

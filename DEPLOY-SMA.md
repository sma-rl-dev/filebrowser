# Deployment & Smoke Test

## Quick Start

```bash
./tester-env deploy   # Build editable source in Docker and start on localhost:8080
./tester-env seed     # Populate deterministic Northwind Design Studio file repository
./tester-env verify   # Check deterministic seed data in the container filesystem
./tester-env status   # Check container state and /health
./tester-env logs     # Follow container logs
./tester-env stop     # Stop container, preserving data volumes
./tester-env reset    # Remove container and config/database/root volumes
```

## Build from Source (required for mutation scenarios)

```bash
# 1. Build frontend
cd frontend
pnpm install
pnpm run build
cd ..

# 2. Build Go backend
go mod download
go build -o filebrowser .

# 3. Generate bcrypt password hash
python3 -c "import bcrypt; print(bcrypt.hashpw(b'adminadminadmin', bcrypt.gensalt()).decode())"

# 4. Run
./filebrowser --password '<bcrypt-hash>'
```

## Credentials

- **Host URL**: http://localhost:8080
- **Browser MCP URL**: http://host.docker.internal:8080
- **Username**: admin
- **Password**: adminadminadmin

The `tester-env` script passes a fixed bcrypt hash for `adminadminadmin` to Filebrowser quick setup so fresh database creation is deterministic.

## Reset

```bash
./tester-env reset
```

Then re-run `./tester-env deploy`.

## Deterministic Seed Data

```bash
./tester-env reset
./tester-env deploy
./tester-env seed
./tester-env verify
```

The seed creates a realistic Northwind Design Studio shared drive under `/srv` with fixed names, contents, and timestamps (`2026-05-21 09:00:00`):

- Root files: `README.md`, `team-roster.csv`
- Root folders: `00_Inbox`, `01_Projects`, `02_Finance`, `03_Archive`, `Client Contracts`
- Nested project files for `Atlas_Redesign` and `Beacon_Mobile_App`, including Markdown notes, JSON endpoint data, and small SVG assets
- Finance records under `02_Finance/2026-Q2`, including `invoices-april.csv` with stable statuses `paid`, `sent`, and `draft`
- Archive and contract records, including `Client Contracts/Nimbus-SOW.txt` with `Status: signed`

Stable verification counts after seeding:

- 17 files, including the hidden `.tester-env-seed-version` marker
- 12 directories, including `/srv`
- 8 first-level entries under `/srv`

`./tester-env seed` is idempotent for its own fixture paths: it removes the known seeded folders/files before recreating them. `./tester-env reset` removes the container and config/database/root volumes, so the next deploy starts with an empty root until seed is run again.

Browser MCP visibility checks should use http://host.docker.internal:8080, log in as `admin` / `adminadminadmin`, and confirm the root listing shows the five root folders plus `README.md` and `team-roster.csv`. Navigate into `01_Projects` -> `Atlas_Redesign` and confirm `brief.md`, `assets`, and `notes` are visible; open `brief.md` and confirm the heading `Atlas Redesign Brief`.

## Smoke Test

1. Open http://host.docker.internal:8080 from browser MCP, or http://localhost:8080 from the host
2. Log in as admin/adminadminadmin
3. Verify the file listing page loads
4. Create a folder via the UI or API
5. Upload a file to the folder
6. Verify the file appears in the listing
7. Delete the file and folder
8. Log out

## Build Notes

- **Must build from source** to create mutation scenarios. The official Docker image (`filebrowser/filebrowser`) is a pre-built binary that cannot be modified.
- Frontend uses `pnpm` + `vite`; backend is Go 1.22+.
- Uses BoltDB (`filebrowser.db`) for user/settings storage.
- Quick setup auto-creates the admin user only when the database is missing; using `--password` skips random password generation.
- The Docker deployment binds Filebrowser to `0.0.0.0:80` in the container and publishes it as `localhost:8080` on the host. Browser MCP runs in a container, so use `http://host.docker.internal:8080` there. Override with `FILEBROWSER_PORT=<port> ./tester-env deploy`.

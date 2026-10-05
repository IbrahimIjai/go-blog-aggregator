# Gator

A multi-user command-line RSS feed aggregator written in Go and backed by PostgreSQL. You can register users, add RSS feeds, follow or unfollow feeds, and fetch their content, all from your terminal.

## Requirements

- Go 1.26+
- PostgreSQL (local install or Docker)
- [goose](https://github.com/pressly/goose) for migrations
- [sqlc](https://sqlc.dev) for generating Go code from SQL

```bash
go install github.com/pressly/goose/v3/cmd/goose@latest
go install github.com/sqlc-dev/sqlc/cmd/sqlc@latest
```

## Setup

1. Start a Postgres database. With Docker:

   ```bash
   docker run -d --name gator-db --restart unless-stopped \
     -p 127.0.0.1:5433:5432 \
     -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=gator \
     -v gator-db-data:/var/lib/postgresql/data postgres:17-alpine
   ```

2. Create `~/.gatorconfig.json`:

   ```json
   {
     "db_url": "postgres://postgres:postgres@localhost:5433/gator?sslmode=disable"
   }
   ```

   `current_user_name` is added automatically when you register or log in.

3. Run the migrations:

   ```bash
   cd sql/schema
   goose postgres "postgres://postgres:postgres@localhost:5433/gator" up
   cd ../..
   ```

## Running

```bash
go run . <command> [args...]
```

Or build a binary:

```bash
go build -o gator .
./gator <command> [args...]
```

| Command | Description |
| --- | --- |
| `register <name>` | Create a user and log in as them |
| `login <name>` | Switch to an existing user |
| `users` | List all users and mark the current one |
| `reset` | Delete all users (and their feeds and follows) |
| `addfeed <name> <url>` | Add a feed and follow it |
| `feeds` | List all feeds and who added them |
| `follow <url>` | Follow an existing feed |
| `following` | List the feeds the current user follows |
| `unfollow <url>` | Unfollow a feed |
| `agg` | Fetch an RSS feed and print it |

## Testing

There are no unit tests yet. Test manually against a clean database:

```bash
go vet ./...
go run . reset
go run . register alice
go run . addfeed "Hacker News RSS" "https://hnrss.org/newest"
go run . following
go run . unfollow "https://hnrss.org/newest"
go run . following
go run . login unknown   # should exit with code 1
```

Check an exit code with `echo $?`. Inspect the data directly with:

```bash
psql "postgres://postgres:postgres@localhost:5433/gator"
```

## Goose (migrations)

Goose runs plain `.sql` migration files in order. Each file lives in `sql/schema`, is named `NNN_name.sql`, and has an `Up` and a `Down` section:

```sql
-- +goose Up
CREATE TABLE users (
    id UUID PRIMARY KEY,
    name TEXT NOT NULL UNIQUE
);

-- +goose Down
DROP TABLE users;
```

Common commands, run from `sql/schema`:

```bash
goose postgres "<connection_string>" up       # apply all pending migrations
goose postgres "<connection_string>" down     # roll back the latest migration
goose postgres "<connection_string>" status   # show which migrations have run
goose create add_posts sql                    # create a new timestamped migration file
```

To add a schema change, create the next numbered file (for example `004_posts.sql`), write its `Up` and `Down`, then run `up`.

## sqlc (queries)

sqlc reads the schema and your SQL queries, then generates type-safe Go code. It's configured in `sqlc.yaml`:

```yaml
version: "2"
sql:
  - schema: "sql/schema"
    queries: "sql/queries"
    engine: "postgresql"
    gen:
      go:
        out: "internal/database"
```

Queries live in `sql/queries`. Each one is annotated with a name and a result type:

```sql
-- name: GetUser :one
SELECT * FROM users WHERE name = $1;
```

- `:one` returns a single row
- `:many` returns a slice of rows
- `:exec` returns only an error
- `:execrows` returns the number of affected rows

Generate the Go code from the project root:

```bash
sqlc generate
```

To start a new project from scratch: run `go mod init`, create `sqlc.yaml`, add your migrations in `sql/schema` and queries in `sql/queries`, then run `sqlc generate`. Use the generated package with `database.New(db)`.

## Project Structure

```
.
├── main.go              entry point and command registration
├── commands.go          command registry
├── middleware.go        logged-in middleware
├── handler_*.go         command handlers
├── rss_feed.go          RSS fetching and parsing
├── internal/
│   ├── config/          reads and writes ~/.gatorconfig.json
│   └── database/        sqlc-generated code (don't edit by hand)
└── sql/
    ├── schema/          goose migrations
    └── queries/         sqlc queries
```

## Notes

- There's no authentication. Anyone with the database credentials can act as any user, so Gator is meant for local use only.
- `reset` wipes all data. Don't run it against a database you care about.
- After changing anything in `sql/`, run the migrations and `sqlc generate` again.

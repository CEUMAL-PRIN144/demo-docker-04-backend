# ToDo API (Express.js)

A REST API for a ToDo app. Todos are stored in a JSON file by default and can be
switched to MySQL by changing one environment variable.

## Quick start

```bash
cd backend
npm install
cp .env.example .env
npm run dev        # or: npm start
```

The API runs on `http://localhost:3000`. Todos are saved to `data/todos.json`.

## Running with Docker

```bash
cd backend
docker build -t todo-backend .
docker run -d --name todo-api -p 3000:3000 -v todo-data:/app/data todo-backend
```

The `todo-data` volume keeps `todos.json` across container restarts. The image
runs as the non-root `node` user and has a health check on `/api/health`.

To use MySQL from the container, pass the settings as environment variables.
`MYSQL_HOST` must be reachable from inside the container. `localhost` there is
the container itself, so use the database's hostname instead. For a MySQL
server on your own machine (Docker Desktop), that's `host.docker.internal`:

```bash
docker run -d -p 3000:3000 \
  -e DATA_SOURCE=mysql \
  -e MYSQL_HOST=host.docker.internal \
  -e MYSQL_USER=root -e MYSQL_PASSWORD=secret -e MYSQL_DATABASE=todo_app \
  todo-backend
```

## Switching to MySQL

1. Set the following in `.env`:
   ```env
   DATA_SOURCE=mysql
   MYSQL_HOST=localhost
   MYSQL_PORT=3306
   MYSQL_USER=root
   MYSQL_PASSWORD=secret
   MYSQL_DATABASE=todo_app
   ```
2. Create the database (`CREATE DATABASE todo_app;`), or run `db/schema.sql`.
   The `todos` table is created automatically on startup.
3. Restart the server. `GET /api/health` reports the data source in use.

### How the switch works

All storage goes through a **repository** with a fixed set of methods:

| Method | Returns |
|---|---|
| `init()` | sets up the file or table |
| `findAll({ completed? })` | array of todos |
| `findById(id)` | todo or `null` |
| `create({ title, description?, completed? })` | created todo |
| `update(id, changes)` | updated todo or `null` |
| `delete(id)` | `true` / `false` |
| `close()` | releases resources |

`src/repositories/index.js` picks the implementation from `DATA_SOURCE`:

- `json.repository.js`: JSON file with serialized, atomic writes
- `mysql.repository.js`: MySQL through a `mysql2` connection pool

Routes only talk to this interface, so to add another backend (Postgres, MongoDB…)
you write one class with these methods and register it in `index.js`.

## Endpoints

| Method | Path | Description |
|---|---|---|
| GET | `/api/health` | Health check + active data source |
| GET | `/api/todos` | List todos (optional `?completed=true\|false`) |
| GET | `/api/todos/:id` | Get one todo |
| POST | `/api/todos` | Create (`title` required; `description`, `completed` optional) |
| PUT | `/api/todos/:id` | Replace (`title` required; omitted fields reset to defaults) |
| PATCH | `/api/todos/:id` | Partial update |
| DELETE | `/api/todos/:id` | Delete (204) |

A todo looks like this:

```json
{
  "id": "4ead1c93-1970-4cca-8eb7-455d31fc932f",
  "title": "Buy milk",
  "description": null,
  "completed": false,
  "createdAt": "2026-11-16T10:47:31.887Z",
  "updatedAt": "2026-11-16T10:47:31.887Z"
}
```

Validation errors return `400` with `{ "error": "...", "details": [...] }`. Unknown IDs return `404`.

### Example

```bash
curl -X POST localhost:3000/api/todos -H 'Content-Type: application/json' -d '{"title":"Buy milk"}'
curl -X PATCH localhost:3000/api/todos/<id> -H 'Content-Type: application/json' -d '{"completed":true}'
```

## Tests

```bash
npm test
```

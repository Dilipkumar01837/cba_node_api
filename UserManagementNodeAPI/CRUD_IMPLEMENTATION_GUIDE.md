# User Management CRUD Notes

These notes explain how the complete User Management CRUD API works and how to run each operation.

## CRUD meaning

- **Create**: Add a new user using `POST`.
- **Read**: Retrieve users using `GET`.
- **Update**: Modify an existing user using `PUT`.
- **Delete**: Remove a user using `DELETE`.

Basic request flow:

```text
Client -> Route -> Controller -> Service -> MySQL -> Response
```

## 1. Project notes

The API uses four layers:

| Layer | Location | Responsibility |
|---|---|---|
| Database | `database/init.sql` | Creates MySQL tables and relationships |
| Models | `models/` | Converts request/database data into domain objects |
| Service | `services/UserService.js` | Executes SQL and applies business rules |
| Controller | `controllers/UserController.js` | Handles HTTP requests and responses |
| Routes | `routes/userRoutes.js` | Maps HTTP methods and URLs to controllers |

The `users` table is related to `address`, `geo`, and `company`. Create, update, and delete operations use transactions so related records remain consistent.

## 2. Setup notes

Install and start:

- Node.js 14 or later
- npm
- MySQL 5.7 or later

Check Node.js and npm:

```bash
node --version
npm --version
```

## 3. Dependency notes

From the project directory:

```bash
cd C:\Training\Node\UserManagementNodeAPI
npm install
```

The project requires Express, MySQL2, CORS, dotenv, express-validator, and nodemon.

## 4. Database notes

Create the database in MySQL:

```sql
CREATE DATABASE IF NOT EXISTS usermanagement;
```

Run the schema script from a terminal:

```bash
mysql -u root -p usermanagement < database/init.sql
```

The script creates these tables:

1. `geo`
2. `address`
3. `company`
4. `users`

`username` and `email` are unique, while foreign keys maintain the relationships between users and their nested records.

## 5. Environment notes

Copy `.env.example` to `.env` and update the values:

```env
PORT=8084
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=your_mysql_password
DB_NAME=usermanagement
```

Do not commit `.env`; it is excluded by `.gitignore`.

## 6. Layer notes

Each request follows this path:

```text
HTTP request
  -> routes/userRoutes.js
  -> validation middleware
  -> controllers/UserController.js
  -> services/UserService.js
  -> MySQL connection pool
  -> JSON response
```

Use parameterized SQL (`?` placeholders) in the service layer to prevent SQL injection.

## 7. Run notes

Start in development mode:

```bash
npm run dev
```

Or start normally:

```bash
npm start
```

Verify the server and database connection:

```bash
curl.exe http://localhost:8084/health
```

Expected response:

```json
{"status":"OK","message":"User Management API is running"}
```

## 8. Create notes — POST

Endpoint: `POST /api/users`

Required fields are `name`, `username`, and a valid `email`. Address, geo, phone, website, and company are optional.

```bash
curl.exe -X POST http://localhost:8084/api/users `
  -H "Content-Type: application/json" `
  -d '{"name":"John Doe","username":"johndoe","email":"john@example.com","phone":"555-0100","website":"example.com","address":{"street":"1 Main Street","suite":"Apt 2","city":"Bengaluru","zipcode":"560001","geo":{"lat":"12.9716","lng":"77.5946"}},"company":{"name":"Example Ltd","catchPhrase":"Simple solutions","bs":"reliable services"}}'
```

The service creates nested records first, stores their IDs in `users`, commits the transaction, and returns the newly created user with HTTP `201`.

Possible results:

- `201`: user created
- `400`: validation failure or duplicate username/email

Save the returned `id` for the next steps.

## 9. Read notes — GET

Get every user:

```bash
curl.exe http://localhost:8084/api/users
```

Get one user by ID:

```bash
curl.exe http://localhost:8084/api/users/1
```

Find by username or email:

```bash
curl.exe http://localhost:8084/api/users/username/johndoe
curl.exe http://localhost:8084/api/users/email/john%40example.com
```

Successful reads return HTTP `200`. A missing user returns HTTP `404`.

## 10. Update notes — PUT

Endpoint: `PUT /api/users/:id`

The current route applies the same validation as create, so send `name`, `username`, and `email` along with any changed values:

```bash
curl.exe -X PUT http://localhost:8084/api/users/1 `
  -H "Content-Type: application/json" `
  -d '{"name":"John Updated","username":"johndoe","email":"john.updated@example.com","phone":"555-0199","website":"updated.example.com"}'
```

The service:

1. Confirms that the user exists.
2. Rejects a username or email already used by another user.
3. Updates supplied address, geo, and company data when present.
4. Updates the user row.
5. Commits all changes together.

Successful updates return HTTP `200`; missing users return `404`; invalid or duplicate data returns `400`.

## 11. Delete notes — DELETE

Endpoint: `DELETE /api/users/:id`

```bash
curl.exe -X DELETE http://localhost:8084/api/users/1
```

The service runs the delete in a transaction, removes the user, and cleans up its address, geo, and company records. A successful delete returns HTTP `200`.

Confirm deletion:

```bash
curl.exe http://localhost:8084/api/users/1
```

Expected result: HTTP `404`.

## 12. Complete CRUD practice

Run these checks in order:

1. `GET /health` — application is running.
2. `GET /api/users` — read the initial collection.
3. `POST /api/users` — create and record the returned ID.
4. `GET /api/users/:id` — confirm the created record.
5. `PUT /api/users/:id` — change a field.
6. `GET /api/users/:id` — confirm the update.
7. `DELETE /api/users/:id` — remove the record.
8. `GET /api/users/:id` — confirm HTTP `404`.

## 13. Important notes

- `ECONNREFUSED`: start MySQL and check `DB_HOST` and `DB_PORT`.
- `Access denied`: correct `DB_USER` and `DB_PASSWORD` in `.env`.
- `Unknown database`: create `usermanagement` before running `database/init.sql`.
- `Username already exists` or `Email already exists`: use unique values.
- `Route not found`: confirm the server uses the `/api/users` prefix.
- `Cannot find module`: run `npm install` from this project directory.

## 14. Future improvement notes

- Add automated unit and integration tests.
- Validate numeric IDs before querying the database.
- Add centralized error handling and request logging.
- Add authentication and authorization before exposing the API publicly.
- Use a secrets manager instead of storing production credentials in `.env`.
- Add API documentation with OpenAPI/Swagger.

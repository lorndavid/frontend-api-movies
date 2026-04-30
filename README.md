# Movie API Frontend

This folder contains the Vue 3 frontend for the Movie REST API.

The backend API is hosted on Render:

```txt
https://backend-api-movies-7r66.onrender.com
```

The frontend runs locally with Vite:

```txt
http://localhost:5173
```

## What This Frontend Does

This frontend is an API console for your backend. It gives you a UI to test the features that currently exist in the Express API.

Features:

1. Check backend health with `GET /hello`
2. Register user with `POST /auth/register`
3. Login user with `POST /auth/login`
4. Logout user with `POST /auth/logout`
5. Test the current `/movies` route
6. Add a movie to watchlist with `POST /watchlist`
7. Update a watchlist item with `PUT /watchlist/:id`
8. Delete a watchlist item with `DELETE /watchlist/:id`

Important: the backend does not yet have a real movie list API. The current `/movies` route only returns test messages. Because of that, the watchlist form asks you to manually paste an existing `movieId`.

## Tech Stack

- Vue 3
- Vite
- Tailwind CSS
- lucide-vue-next icons
- Render backend API

## Folder Structure

```txt
frontend/
  index.html
  package.json
  vite.config.js
  tailwind.config.js
  postcss.config.js
  .env
  .env.example
  src/
    App.vue
    api.js
    main.js
    styles.css
```

Main files:

```txt
src/App.vue
```

This is the main UI. It contains the forms and buttons for auth, movies, and watchlist.

```txt
src/api.js
```

This is the small API helper. It sends requests to the backend.

```txt
vite.config.js
```

This contains the local development proxy that fixes CORS problems while testing on your computer.

## How Frontend And Backend Work Together

Your browser opens the frontend:

```txt
http://localhost:5173
```

The frontend needs to call the backend:

```txt
https://backend-api-movies-7r66.onrender.com
```

But browsers have a security rule called CORS. CORS can block requests from one domain to another domain.

Example:

```txt
Frontend origin:
http://localhost:5173

Backend origin:
https://backend-api-movies-7r66.onrender.com
```

Those are different origins, so the browser checks if the backend allows the frontend.

If the backend does not return this header:

```txt
Access-Control-Allow-Origin: http://localhost:5173
```

the browser blocks the request.

That is why you saw this error:

```txt
No 'Access-Control-Allow-Origin' header is present on the requested resource
```

## Why This Project Uses `/api`

For local development, the frontend uses:

```env
VITE_API_BASE_URL=/api
```

That means the browser calls:

```txt
http://localhost:5173/api/hello
```

not directly:

```txt
https://backend-api-movies-7r66.onrender.com/hello
```

Then Vite forwards the request to Render.

This is configured in `vite.config.js`:

```js
server: {
  proxy: {
    "/api": {
      target: "https://backend-api-movies-7r66.onrender.com",
      changeOrigin: true,
      rewrite: (path) => path.replace(/^\/api/, ""),
    },
  },
}
```

So this local request:

```txt
http://localhost:5173/api/hello
```

becomes this backend request:

```txt
https://backend-api-movies-7r66.onrender.com/hello
```

This avoids local CORS problems because the browser only talks to Vite on `localhost:5173`.

## Environment Variables

The local `.env` file should contain:

```env
VITE_API_BASE_URL=/api
```

Use `/api` for local development.

The `.env` file is ignored by Git, so it should not be pushed to GitHub.

There is also an example file:

```txt
.env.example
```

## Install And Run

From the project root:

```bash
cd frontend
npm install
npm run dev
```

Open:

```txt
http://localhost:5173
```

## Build For Production

Run:

```bash
npm run build
```

This creates:

```txt
frontend/dist
```

The `dist` folder is the production frontend output.

## Preview Production Build Locally

After building:

```bash
npm run preview
```

Open the preview URL that Vite shows.

## How To Test The UI

### 1. Check Backend

Click:

```txt
Check
```

The frontend calls:

```txt
GET /hello
```

Expected response:

```json
{
  "message": "Hello, World!"
}
```

### 2. Register User

Use the Register tab.

Example:

```json
{
  "name": "David",
  "email": "david@example.com",
  "password": "123456"
}
```

The frontend calls:

```txt
POST /auth/register
```

If successful, the backend returns a JWT token. The frontend saves that token in browser local storage.

### 3. Login User

Use the Login tab.

Example:

```json
{
  "email": "david@example.com",
  "password": "123456"
}
```

The frontend calls:

```txt
POST /auth/login
```

If successful, the frontend saves the JWT token.

### 4. Test Movie Route

The UI can test:

```txt
GET /movies
POST /movies
PUT /movies
DELETE /movies
```

Right now your backend movie route is only a test route. It does not yet create or list real movies.

### 5. Add To Watchlist

This request requires login.

The frontend sends the JWT token in the request header:

```txt
Authorization: Bearer YOUR_TOKEN
```

Body example:

```json
{
  "movieId": "existing-movie-uuid",
  "status": "PLANNED",
  "rating": 8,
  "notes": "I want to watch this later"
}
```

The backend route:

```txt
POST /watchlist
```

The `movieId` must already exist in the database.

If you use Neon, open the Neon SQL editor and run:

```sql
SELECT id, title FROM "Movie";
```

Copy one `id` and paste it into the frontend.

### 6. Update Watchlist Item

After adding to watchlist, copy the returned watchlist item ID.

The frontend calls:

```txt
PUT /watchlist/:id
```

Body example:

```json
{
  "status": "COMPLETED",
  "rating": 10,
  "notes": "Finished watching"
}
```

### 7. Delete Watchlist Item

The frontend calls:

```txt
DELETE /watchlist/:id
```

You must be logged in, and the watchlist item must belong to the logged-in user.

## How Authentication Works

When login or register succeeds, the backend sends:

```json
{
  "token": "jwt-token-here"
}
```

The frontend saves the token in local storage:

```txt
movie-api-session
```

For protected routes, the frontend sends:

```txt
Authorization: Bearer jwt-token-here
```

The backend middleware checks the token and sets:

```js
req.user
```

Then the watchlist controller uses:

```js
req.user.id
```

to know which user owns the watchlist item.

## Important CORS Note For Deployment

The Vite proxy only works in local development.

When you deploy the frontend to a real domain, for example:

```txt
https://movie-api-frontend.onrender.com
```

the backend must allow that frontend domain.

Set this environment variable on the backend Render service:

```env
CLIENT_ORIGIN=https://movie-api-frontend.onrender.com,http://localhost:5173,http://127.0.0.1:5173
```

Then redeploy the backend.

## Deploy Frontend To Render

Create a new Render Static Site.

Settings:

```txt
Root Directory: frontend
Build Command: npm install && npm run build
Publish Directory: dist
```

Environment variable for production:

```env
VITE_API_BASE_URL=https://backend-api-movies-7r66.onrender.com
```

After deploying the frontend, copy the frontend Render URL and add it to the backend `CLIENT_ORIGIN` environment variable.

Example:

```env
CLIENT_ORIGIN=https://your-frontend-name.onrender.com
```

Then redeploy the backend.

## Common Problems

### CORS Error

Error:

```txt
No 'Access-Control-Allow-Origin' header is present
```

Local fix:

```env
VITE_API_BASE_URL=/api
```

Then restart Vite:

```bash
npm run dev
```

Production fix:

Add frontend domain to backend `CLIENT_ORIGIN`, then redeploy backend.

### Backend Shows Unavailable

Render free services can sleep when inactive. The first request can take time.

Click `Check` again after 30-60 seconds.

### Register Returns 400

Possible reasons:

1. Email already exists.
2. Missing name, email, or password.
3. Backend environment variable `JWT_SECRET` is missing.

Try a new email:

```txt
david2@example.com
```

### Watchlist Says Movie Not Found

The `movieId` does not exist in the database.

Check Neon:

```sql
SELECT id, title FROM "Movie";
```

Then paste an existing movie ID.

## Useful Commands

Run development server:

```bash
npm run dev
```

Build frontend:

```bash
npm run build
```

Preview build:

```bash
npm run preview
```

Install packages:

```bash
npm install
```

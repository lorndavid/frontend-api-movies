<script setup>
import { computed, onMounted, reactive, ref } from "vue";
import {
  Activity,
  AlertCircle,
  CheckCircle2,
  Film,
  KeyRound,
  ListPlus,
  LogIn,
  LogOut,
  Pencil,
  RefreshCw,
  Send,
  Server,
  ShieldCheck,
  Trash2,
  UserPlus,
} from "lucide-vue-next";
import { API_BASE_URL, apiRequest } from "./api";

const session = ref(loadSession());
const authMode = ref("register");
const loading = reactive({
  health: false,
  auth: false,
  movies: false,
  addWatchlist: false,
  updateWatchlist: false,
  deleteWatchlist: false,
});

const authForm = reactive({
  name: "David",
  email: "david@example.com",
  password: "123456",
});

const movieMethod = ref("GET");
const watchlistForm = reactive({
  movieId: "",
  status: "PLANNED",
  rating: 8,
  notes: "I want to watch this later.",
});

const watchlistEditForm = reactive({
  id: "",
  status: "COMPLETED",
  rating: 10,
  notes: "Finished watching.",
});

const health = reactive({
  checked: false,
  ok: false,
  message: "Not checked",
});

const lastResponse = reactive({
  title: "Response",
  status: null,
  ok: null,
  data: {
    ready: true,
    api: API_BASE_URL,
  },
});

const isAuthenticated = computed(() => Boolean(session.value?.token));
const sessionEmail = computed(() => session.value?.user?.email || "Guest");

function loadSession() {
  try {
    return JSON.parse(localStorage.getItem("movie-api-session"));
  } catch {
    return null;
  }
}

function saveSession(nextSession) {
  session.value = nextSession;
  localStorage.setItem("movie-api-session", JSON.stringify(nextSession));
}

function clearSession() {
  session.value = null;
  localStorage.removeItem("movie-api-session");
}

function setResponse(title, result) {
  lastResponse.title = title;
  lastResponse.status = result.status;
  lastResponse.ok = result.ok;
  lastResponse.data = result.data;
}

async function runAction(key, title, action) {
  loading[key] = true;

  try {
    const result = await action();
    setResponse(title, result);
    return result;
  } catch (error) {
    const result = {
      ok: false,
      status: "NETWORK",
      data: {
        error: error.message || "Request failed",
      },
    };
    setResponse(title, result);
    return result;
  } finally {
    loading[key] = false;
  }
}

async function checkHealth() {
  const result = await runAction("health", "GET /hello", () =>
    apiRequest("/hello"),
  );

  health.checked = true;
  health.ok = result.ok;
  health.message =
    typeof result.data === "object" && result.data?.message
      ? result.data.message
      : result.ok
        ? "Online"
        : "Unavailable";
}

async function submitAuth() {
  const path = authMode.value === "register" ? "/auth/register" : "/auth/login";
  const body =
    authMode.value === "register"
      ? { ...authForm }
      : { email: authForm.email, password: authForm.password };

  const result = await runAction("auth", `POST ${path}`, () =>
    apiRequest(path, {
      method: "POST",
      body,
    }),
  );

  const token = result.data?.data?.token;
  const user = result.data?.data?.user;

  if (result.ok && token) {
    saveSession({ token, user });
  }
}

async function logout() {
  await runAction("auth", "POST /auth/logout", () =>
    apiRequest("/auth/logout", {
      method: "POST",
    }),
  );
  clearSession();
}

async function testMovieRoute() {
  await runAction("movies", `${movieMethod.value} /movies`, () =>
    apiRequest("/movies", {
      method: movieMethod.value,
    }),
  );
}

async function addWatchlistItem() {
  const result = await runAction("addWatchlist", "POST /watchlist", () =>
    apiRequest("/watchlist", {
      method: "POST",
      token: session.value?.token,
      body: {
        movieId: watchlistForm.movieId.trim(),
        status: watchlistForm.status,
        rating: Number(watchlistForm.rating),
        notes: watchlistForm.notes,
      },
    }),
  );

  const itemId = result.data?.data?.watchlistItem?.id;
  if (result.ok && itemId) {
    watchlistEditForm.id = itemId;
  }
}

async function updateWatchlistItem() {
  await runAction("updateWatchlist", `PUT /watchlist/${watchlistEditForm.id}`, () =>
    apiRequest(`/watchlist/${watchlistEditForm.id.trim()}`, {
      method: "PUT",
      token: session.value?.token,
      body: {
        status: watchlistEditForm.status,
        rating: Number(watchlistEditForm.rating),
        notes: watchlistEditForm.notes,
      },
    }),
  );
}

async function deleteWatchlistItem() {
  const id = watchlistEditForm.id.trim();

  const result = await runAction("deleteWatchlist", `DELETE /watchlist/${id}`, () =>
    apiRequest(`/watchlist/${id}`, {
      method: "DELETE",
      token: session.value?.token,
    }),
  );

  if (result.ok) {
    watchlistEditForm.id = "";
  }
}

onMounted(checkHealth);
</script>

<template>
  <main class="mx-auto flex min-h-screen w-full max-w-7xl flex-col gap-6 px-4 py-5 sm:px-6 lg:px-8">
    <header class="flex flex-col gap-4 rounded-lg border border-zinc-200 bg-white px-5 py-4 shadow-panel lg:flex-row lg:items-center lg:justify-between">
      <div class="flex min-w-0 items-center gap-4">
        <div class="flex size-12 shrink-0 items-center justify-center rounded-lg bg-zinc-950 text-white">
          <Film :size="24" aria-hidden="true" />
        </div>
        <div class="min-w-0">
          <h1 class="text-2xl font-semibold tracking-normal text-zinc-950">
            Movie API Console
          </h1>
          <p class="truncate text-sm text-zinc-500">
            {{ API_BASE_URL }}
          </p>
        </div>
      </div>

      <div class="flex flex-wrap items-center gap-2">
        <span
          class="badge"
          :class="health.ok ? 'border-emerald-200 bg-emerald-50 text-emerald-700' : 'border-amber-200 bg-amber-50 text-amber-700'"
        >
          <CheckCircle2 v-if="health.ok" :size="16" aria-hidden="true" />
          <AlertCircle v-else :size="16" aria-hidden="true" />
          {{ health.message }}
        </span>
        <span class="badge border-zinc-200 bg-zinc-50 text-zinc-700">
          <ShieldCheck :size="16" aria-hidden="true" />
          {{ sessionEmail }}
        </span>
        <button class="btn btn-soft" type="button" :disabled="loading.health" @click="checkHealth">
          <RefreshCw :size="17" aria-hidden="true" />
          Check
        </button>
      </div>
    </header>

    <section class="grid gap-6 lg:grid-cols-[minmax(0,0.95fr)_minmax(360px,0.75fr)]">
      <div class="grid gap-6">
        <section class="panel">
          <div class="panel-header">
            <div class="flex items-center gap-3">
              <div class="flex size-10 items-center justify-center rounded-md bg-emerald-50 text-emerald-700">
                <KeyRound :size="20" aria-hidden="true" />
              </div>
              <div>
                <h2 class="text-base font-semibold text-zinc-950">Account</h2>
                <p class="text-sm text-zinc-500">Register, login, and token session.</p>
              </div>
            </div>
            <button v-if="isAuthenticated" class="btn btn-soft" type="button" @click="logout">
              <LogOut :size="17" aria-hidden="true" />
              Logout
            </button>
          </div>

          <div class="grid gap-5 p-5">
            <div class="grid grid-cols-2 gap-2 rounded-md bg-zinc-100 p-1">
              <button
                class="btn h-10"
                :class="authMode === 'register' ? 'bg-white text-zinc-950 shadow-sm' : 'text-zinc-600 hover:text-zinc-950'"
                type="button"
                @click="authMode = 'register'"
              >
                <UserPlus :size="17" aria-hidden="true" />
                Register
              </button>
              <button
                class="btn h-10"
                :class="authMode === 'login' ? 'bg-white text-zinc-950 shadow-sm' : 'text-zinc-600 hover:text-zinc-950'"
                type="button"
                @click="authMode = 'login'"
              >
                <LogIn :size="17" aria-hidden="true" />
                Login
              </button>
            </div>

            <form class="grid gap-4 md:grid-cols-2" @submit.prevent="submitAuth">
              <label v-if="authMode === 'register'" class="field md:col-span-2">
                <span class="label">Name</span>
                <input v-model="authForm.name" class="input" type="text" autocomplete="name" />
              </label>

              <label class="field">
                <span class="label">Email</span>
                <input v-model="authForm.email" class="input" type="email" autocomplete="email" />
              </label>

              <label class="field">
                <span class="label">Password</span>
                <input v-model="authForm.password" class="input" type="password" autocomplete="current-password" />
              </label>

              <div class="md:col-span-2">
                <button class="btn btn-primary w-full sm:w-auto" type="submit" :disabled="loading.auth">
                  <Send :size="17" aria-hidden="true" />
                  {{ authMode === "register" ? "Create account" : "Login" }}
                </button>
              </div>
            </form>
          </div>
        </section>

        <section class="panel">
          <div class="panel-header">
            <div class="flex items-center gap-3">
              <div class="flex size-10 items-center justify-center rounded-md bg-sky-50 text-sky-700">
                <Server :size="20" aria-hidden="true" />
              </div>
              <div>
                <h2 class="text-base font-semibold text-zinc-950">Movie Routes</h2>
                <p class="text-sm text-zinc-500">GET, POST, PUT, and DELETE `/movies`.</p>
              </div>
            </div>
          </div>

          <div class="grid gap-5 p-5">
            <div class="grid grid-cols-2 gap-2 sm:grid-cols-4">
              <button
                v-for="method in ['GET', 'POST', 'PUT', 'DELETE']"
                :key="method"
                class="btn h-10"
                :class="movieMethod === method ? 'bg-zinc-950 text-white' : 'border border-zinc-300 bg-white text-zinc-800 hover:bg-zinc-100'"
                type="button"
                @click="movieMethod = method"
              >
                {{ method }}
              </button>
            </div>

            <button class="btn btn-primary w-full sm:w-auto" type="button" :disabled="loading.movies" @click="testMovieRoute">
              <Activity :size="17" aria-hidden="true" />
              Send request
            </button>
          </div>
        </section>

        <section class="panel">
          <div class="panel-header">
            <div class="flex items-center gap-3">
              <div class="flex size-10 items-center justify-center rounded-md bg-amber-50 text-amber-700">
                <ListPlus :size="20" aria-hidden="true" />
              </div>
              <div>
                <h2 class="text-base font-semibold text-zinc-950">Watchlist</h2>
                <p class="text-sm text-zinc-500">Protected add, update, and remove requests.</p>
              </div>
            </div>
            <span
              class="badge"
              :class="isAuthenticated ? 'border-emerald-200 bg-emerald-50 text-emerald-700' : 'border-rose-200 bg-rose-50 text-rose-700'"
            >
              {{ isAuthenticated ? "Token ready" : "Login required" }}
            </span>
          </div>

          <div class="grid gap-6 p-5 xl:grid-cols-2">
            <form class="grid gap-4" @submit.prevent="addWatchlistItem">
              <div class="flex items-center gap-2 text-sm font-semibold text-zinc-950">
                <ListPlus :size="18" aria-hidden="true" />
                Add item
              </div>

              <label class="field">
                <span class="label">Movie ID</span>
                <input v-model="watchlistForm.movieId" class="input" type="text" placeholder="UUID from Movie table" />
              </label>

              <div class="grid gap-4 sm:grid-cols-2">
                <label class="field">
                  <span class="label">Status</span>
                  <select v-model="watchlistForm.status" class="input">
                    <option>PLANNED</option>
                    <option>WATCHING</option>
                    <option>COMPLETED</option>
                    <option>DROPPED</option>
                  </select>
                </label>

                <label class="field">
                  <span class="label">Rating</span>
                  <input v-model.number="watchlistForm.rating" class="input" type="number" min="1" max="10" />
                </label>
              </div>

              <label class="field">
                <span class="label">Notes</span>
                <textarea v-model="watchlistForm.notes" class="textarea" />
              </label>

              <button class="btn btn-primary w-full sm:w-auto" type="submit" :disabled="!isAuthenticated || loading.addWatchlist">
                <ListPlus :size="17" aria-hidden="true" />
                Add to watchlist
              </button>
            </form>

            <form class="grid gap-4" @submit.prevent="updateWatchlistItem">
              <div class="flex items-center gap-2 text-sm font-semibold text-zinc-950">
                <Pencil :size="18" aria-hidden="true" />
                Update item
              </div>

              <label class="field">
                <span class="label">Watchlist Item ID</span>
                <input v-model="watchlistEditForm.id" class="input" type="text" />
              </label>

              <div class="grid gap-4 sm:grid-cols-2">
                <label class="field">
                  <span class="label">Status</span>
                  <select v-model="watchlistEditForm.status" class="input">
                    <option>PLANNED</option>
                    <option>WATCHING</option>
                    <option>COMPLETED</option>
                    <option>DROPPED</option>
                  </select>
                </label>

                <label class="field">
                  <span class="label">Rating</span>
                  <input v-model.number="watchlistEditForm.rating" class="input" type="number" min="1" max="10" />
                </label>
              </div>

              <label class="field">
                <span class="label">Notes</span>
                <textarea v-model="watchlistEditForm.notes" class="textarea" />
              </label>

              <div class="flex flex-col gap-2 sm:flex-row">
                <button class="btn btn-primary" type="submit" :disabled="!isAuthenticated || !watchlistEditForm.id || loading.updateWatchlist">
                  <Pencil :size="17" aria-hidden="true" />
                  Update
                </button>
                <button class="btn btn-danger" type="button" :disabled="!isAuthenticated || !watchlistEditForm.id || loading.deleteWatchlist" @click="deleteWatchlistItem">
                  <Trash2 :size="17" aria-hidden="true" />
                  Delete
                </button>
              </div>
            </form>
          </div>
        </section>
      </div>

      <aside class="grid content-start gap-6">
        <section class="panel">
          <div class="panel-header">
            <div class="flex items-center gap-3">
              <div class="flex size-10 items-center justify-center rounded-md bg-zinc-100 text-zinc-800">
                <Activity :size="20" aria-hidden="true" />
              </div>
              <div>
                <h2 class="text-base font-semibold text-zinc-950">{{ lastResponse.title }}</h2>
                <p class="text-sm text-zinc-500">
                  Status {{ lastResponse.status ?? "-" }}
                </p>
              </div>
            </div>
            <span
              class="badge"
              :class="lastResponse.ok === null ? 'border-zinc-200 bg-zinc-50 text-zinc-700' : lastResponse.ok ? 'border-emerald-200 bg-emerald-50 text-emerald-700' : 'border-rose-200 bg-rose-50 text-rose-700'"
            >
              {{ lastResponse.ok === null ? "Idle" : lastResponse.ok ? "OK" : "Error" }}
            </span>
          </div>

          <pre class="max-h-[620px] overflow-auto p-5 text-sm leading-6 text-zinc-800">{{ JSON.stringify(lastResponse.data, null, 2) }}</pre>
        </section>
      </aside>
    </section>
  </main>
</template>

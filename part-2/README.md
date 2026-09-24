# Part 2: Docker Compose

Now that we have a production-ready version of the frontend image, let's deploy the frontend and backend together using Docker Compose.

## Motivation

In Part 1, we ended up with a single, production-ready Docker image for the frontend. But a frontend alone doesn't make an app — we still need a backend (and, as we'll see, a database) running alongside it, wired together with the right ports, environment variables, and networking. Running each of these by hand with `docker run` means remembering a growing pile of flags every time, and none of it is written down anywhere. Docker Compose lets us describe the whole stack — every service, port, and environment variable — in one YAML file, and bring it all up with a single command.

## Getting Started with Docker Compose

We've provided a starter [`docker-compose.yml`](./docker-compose.yml) with the `frontend` service already configured:

```yaml
services:
  frontend:
    build:
      context: ../part-1/frontend
      dockerfile: ../Dockerfile
      args:
        VITE_BACKEND_BASE_URL: http://localhost:8000
- Instead of pulling a prebuilt image, `build:` tells Compose to build the image straight from the Dockerfile you wrote in Part 1. Compose will rebuild it automatically whenever you run `docker compose up --build`.
    ports:
      - "8080:80"
```

A couple of things worth noticing here:

- `VITE_BACKEND_BASE_URL` is passed in under `args:`, not `environment:`. Recall from Part 1 that this value gets baked into the static bundle at _build_ time — by the time the container is running, there's no process left to read a runtime environment variable, so `environment:` wouldn't do anything here.
- The URL itself is `http://localhost:8000`, not something like `httpn://backend:8000`. That's because this URL is used by your _browser_, not by a container — and your browser only knows about `localhost`, not Compose's internal network.

### Configuring the Backend Service

Add a `backend` service to `docker-compose.yml`, using the `cis1912/hw01-backend` image. By default it exposes port `8000`; publish it to the host so both the frontend (from your browser) and you (for testing) can reach it.

## Testing the Integration

Bring the whole stack up:

```bash
docker compose up --build
```

Once both containers are running, open `http://localhost:8080` in your browser and verify that:

- You can create a new todo
- Todos appear in the list
- You can check off or delete a todo

If something isn't working, `docker compose logs backend` and `docker compose logs frontend` will show you what each container is doing.

## Adding Redis

Right now, the backend keeps todos in memory. Try it out: create a few todos, then restart just the backend container:

```bash
docker compose restart backend
```

Refresh the frontend — your todos are gone, because in-memory storage lives inside the container's process and gets wiped out on restart.

`ghcr.io/cis1912/hw01-backend` already supports persisting tasks in [Redis](https://redis.io/) instead — at startup, it checks for a `REDIS_URL` environment variable and uses Redis if it's set, falling back to in-memory storage otherwise. All that's left is to give it a Redis instance to talk to, purely through `docker-compose.yml`:

1. Add a new `redis` service, using the official `redis` image.
2. Add a `REDIS_URL` environment variable to `backend` that points at your new `redis` service.

A couple of hints:

- Compose places every service defined in the same file onto a shared network by default, and each service can reach the others **by service name** — so if you name your service `redis`, the backend can reach it at host `redis`, no IP address needed.
- Redis listens on port `6379` by default, and the backend expects a URL of the form `redis://<host>:<port>/<db>`.

Bring the stack up again, create a few todos, and restart the backend the same way as before. This time your todos should survive the restart — proof that Redis, not the backend's memory, is now the source of truth.

## Health Checks

Try the following experiment a few times: tear the whole stack down and bring it back up from scratch.

```bash
docker compose down
docker compose up --build
```

Depending on timing, you might notice the backend occasionally fails to persist your first few todos to Redis, even though `REDIS_URL` is set correctly. What's going on?

Compose starts containers, but starting a container isn't the same as the service inside it being _ready_. The `redis` image takes a moment to initialize before it can actually accept connections — and if the backend tries to connect before that happens, it may silently fall back to in-memory storage for good, even once Redis becomes reachable a second later.

Docker (and Compose) let you define a [`healthcheck`](https://docs.docker.com/reference/compose-file/services/#healthcheck) for a service: a command that gets run periodically inside the container to determine whether the service is actually ready to do its job, not just "started." A service isn't considered `healthy` until its healthcheck passes.

Add a `healthcheck` to your `redis` service. Redis ships with a CLI you can use to check whether it's accepting connections — look into what command that might be, and how to phrase it as a Compose `healthcheck`. Run `docker compose ps` after bringing the stack up to confirm your `redis` service reports as `healthy` and not just `running`.

> 💡 Once you've got `redis` reporting healthy, think about whether `backend` needs a `healthcheck` of its own too, for the step below.

## Waiting for Dependencies

A `healthcheck` alone doesn't change _when_ Compose starts a service — for that, we need `depends_on`. You may already be relying on plain `depends_on` (or the implicit ordering from referencing a service by name) to make sure `redis` and `backend` start in the right order. But by default, `depends_on` only waits for a dependency's container to _start_ — not for it to be healthy. That's exactly the gap that caused the flaky behavior you saw above.

Compose's `depends_on` supports an extended form that lets you wait on a specific _condition_, including a dependency reporting healthy, rather than just started. Look into how to express this in your `docker-compose.yml`, and use it so that:

- `backend` doesn't start until `redis` is healthy
- `frontend` doesn't start until `backend` is healthy

Tear everything down and bring it back up a few times to confirm the flakiness from before is gone.

## Isolating the Network

At this point, `redis` is reachable by `backend` — but on Compose's default shared network, it's _also_ reachable by `frontend`, and by anything else you might add to this file later. And if you happened to publish Redis's port to the host, anyone on your machine could connect straight to it with `redis-cli` or any other Redis client, completely bypassing your backend's API.

That's a problem: the whole point of building a backend API is to control how data gets read and written. If any client can reach Redis directly, they can read, modify, or delete tasks without going through the validation and logic your backend defines.

Docker Compose lets you define custom [networks](https://docs.docker.com/compose/how-tos/networking/) and control exactly which services can reach each other. Let's use this so that only `backend` can reach `redis` — not `frontend`, and not anything outside of Compose.

1. Make sure you have **not** published Redis's port to the host (remove any `ports:` entry under the `redis` service if you added one).
2. Define a custom network (e.g. `backend-net`) under a top-level `networks:` key in your `docker-compose.yml`.
3. Attach both `backend` and `redis` to `backend-net`, using a `networks:` key under each of those two services.
4. Leave `frontend` off of `backend-net` — it only ever needs to talk to `backend`'s API, never to Redis directly.

> ⚠️ Once a service declares an explicit `networks:` key, it stops joining Compose's default network automatically. Make sure `backend` can still be reached by `frontend` after this change — you may need to list more than one network under `backend`.

Once you've made these changes, bring the stack up again and confirm:

- The app still works end-to-end (frontend → backend → Redis) at `http://localhost:8080`.
- You can no longer reach Redis directly from your host machine — e.g. `redis-cli -h localhost -p 6379 ping` should fail to connect.

**Why bother?** Isolating `redis` onto its own internal network prevents external connections from directly connecting to the Redis instance, which breaks our goal of abstraction / encapsulation — the backend, and only the backend, should get to decide how the underlying data store is read from and written to.

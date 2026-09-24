# Part 1: Finishing the Production Frontend Deployment

Recall from Lab 1 that we put together a working frontend Docker image for the Todo app, using the `yarn dev` command to launch the server. But wait — `dev` sounds a bit suspicious! `yarn dev` starts a _development_ server, which comes with extras like hot reloading and verbose debugging on top of the basic server functionality. In production, those extras are just unnecessary overhead, and sometimes even a security liability — not something we want to ship.

The correct way to run a frontend in production is to serve its **build output**, not the dev server. Modern frontend tooling (Vite, in our case) compiles your source into a small set of static, optimized files. A lightweight static file server — such as `Caddy`, `nginx`, or `traefik` — can then host those files directly. For this assignment, we'll use `Caddy`.

So we need two things:

- A consistent way to build the production frontend bundle
- A consistent way to serve the bundle built in the previous step

We'll do this by using something called a _multi-stage build_.

## Step 0: What is a multi-stage build?

> 🤔 This is something we'd like you to think about...

In [RESPONSE.md](../RESPONSE.md), write your response to the following question. You'll find the [Docker docs](https://docs.docker.com/build/building/multi-stage/) on this topic useful.

> What is a multi-stage build, and why is it necessary? How does it help us build the Todo frontend?

## Step 1: The Builder Stage

We've provided a working version of the frontend code in `frontend/`. Note that `frontend/` is _not_ in the same directory as the [Dockerfile](./Dockerfile) — keep this in mind for any host-side paths you reference (e.g. build context, `COPY` sources).

The Dockerfile already sketches out the two stages you need: a builder stage (`node:20-slim`) and a runner stage (`caddy:alpine`). Your job in this step is to fill in the **builder** stage so that it installs the frontend's dependencies and produces a production build.

A couple of things worth remembering from Lab 1:

- Copying only the dependency manifests (`package.json`, `yarn.lock`) before running the install command lets Docker cache that layer, so you're not reinstalling everything every time your source changes.
- The builder stage is never actually run as a container — it only exists to produce files that the next stage can copy. Keep that in mind when deciding what needs to happen here versus in the runner stage.

## Step 2: The Runner Stage

Now that the builder produces a production build, the runner stage needs to "resume" the job: take the build output from the builder stage and copy it into the `caddy:alpine` image so Caddy can serve it. The build output from the frontend should live in `dist`.

Caddy's official image already ships with a default configuration that serves static files out of the box — you shouldn't need to write any custom Caddy configuration for this assignment. Take a look at the [Caddy image on Docker Hub](https://hub.docker.com/_/caddy) to figure out where it expects static files to live.

## Step 3: Testing

Once both stages are filled in, build and run the image. Remember that the build context needs to point at `frontend/`, while the Dockerfile itself lives in `part-1/`:

```bash
docker build -f Dockerfile -t todo-frontend-prod frontend
docker run -p 8080:80 todo-frontend-prod
```

Visit `http://localhost:8080` in your browser — you should see the Todo app. If something looks wrong, `docker logs <container>` will show you what Caddy is doing.

Keep in mind the builder stage always runs before the runner stage — the builder's only job is to prepare the artifacts that get handed off. Once this is working, move on to Step 4.

## Step 4: Addressing the ENV Issue

You may have noticed that we've now lost the ability to set the backend URL, `VITE_BACKEND_BASE_URL`, at runtime the way `docker run -e ...` let us do in Lab 1. This is expected for a production build: Vite bakes environment variables into the static bundle _at build time_, so by the time Caddy is serving the files, there's no process left to read a runtime environment variable from.

Docker gives us a way to set values like this at build time instead, using `ARG`. Add a build argument named `VITE_BACKEND_BASE_URL` to your builder stage, and make sure it's available to the `yarn build` step (think about how `ARG` and `ENV` interact within a stage).

Once that's wired up, you can pass the value in at build time:

```bash
docker build -f Dockerfile -t todo-frontend-prod --build-arg VITE_BACKEND_BASE_URL=http://localhost:8000 frontend
docker run -p 8080:80 todo-frontend-prod
```

Verify that the app at `http://localhost:8080` can now reach a backend running on `http://localhost:8000`.

## Tips

- `.dockerignore` keeps your build context lean and prevents accidentally copying local files (like a host-installed `node_modules/`) into the image — make sure yours excludes what it should.
- Compare `docker images` before and after switching from the dev-server Dockerfile to this multi-stage one. The size difference is the whole point of not shipping your build tooling in the final image.
- If `docker build` succeeds but the page doesn't load, double check you're publishing the right container port and that the files ended up where Caddy expects them.

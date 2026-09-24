FROM node:20-slim AS builder

# Build the frontend

WORKDIR /app
COPY package.json yarn.lock ./

RUN yarn install --frozen-lockfile

COPY . .

ARG VITE_BACKEND_BASE_URL
ENV VITE_BACKEND_BASE_URL=$VITE_BACKEND_BASE_URL

RUN yarn build

FROM caddy:alpine

# Copy the build output from the builder image
COPY --from=builder /app/dist /usr/share/caddy


EXPOSE 80

CMD ["caddy", "run", "--config", "/etc/caddy/Caddyfile", "--adapter", "caddyfile"]

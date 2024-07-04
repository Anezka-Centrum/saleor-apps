FROM node:18-alpine AS alpine

# Setup pnpm on the alpine base
FROM alpine as base
ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME:$PATH"
RUN corepack enable
RUN pnpm install turbo --global

FROM base AS builder
# Check https://github.com/nodejs/docker-node/tree/b4117f9333da4138b03a546ec926ef50a31506c3#nodealpine to understand why libc6-compat might be needed.
RUN apk add --no-cache libc6-compat
RUN apk update
# Set working directory
WORKDIR /app
COPY . .

RUN turbo prune --scope=typesense-app --docker

# Add lockfile and package.json's of isolated subworkspace
FROM base AS installer
RUN apk add --no-cache libc6-compat
RUN apk update
WORKDIR /app

# First install the dependencies (as they change less often)
COPY .gitignore .gitignore
COPY --from=builder /app/out/json/ .
COPY --from=builder /app/out/pnpm-lock.yaml ./pnpm-lock.yaml
COPY --from=builder /app/out/pnpm-workspace.yaml ./pnpm-workspace.yaml
RUN pnpm install

# Build the project
COPY --from=builder /app/out/full/ .
COPY turbo.json turbo.json

# Uncomment and use build args to enable remote caching
# ARG TURBO_TEAM
# ENV TURBO_TEAM=$TURBO_TEAM

# ARG TURBO_TOKEN
# ENV TURBO_TOKEN=$TURBO_TOKEN

RUN turbo run build --filter=typesense-app

# Use alpine as the thinnest image

FROM alpine AS runner

WORKDIR /app

# Don't run production as root
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs
USER nextjs

COPY --from=installer /app/apps/typesense-app/next.config.js .
COPY --from=installer /app/apps/typesense-app/package.json .

# Automatically leverage output traces to reduce image size
# https://nextjs.org/docs/advanced-features/output-file-tracing
COPY --from=installer --chown=nextjs:nodejs /app/apps/typesense-app/.next/standalone ./
COPY --from=installer --chown=nextjs:nodejs /app/apps/typesense-app/.next/static ./apps/typesense-app/.next/static
COPY --from=installer --chown=nextjs:nodejs /app/apps/typesense-app/public ./apps/typesense-app/public

# Set workdir to the app name 
WORKDIR /app/apps/typesense-app

# Pass environment variables

ARG SECRET_KEY
ARG APP_IFRAME_BASE_URL
ARG APP_API_BASE_URL

ENV SECRET_KEY=${SECRET_KEY}
ENV APP_IFRAME_BASE_URL=${APP_IFRAME_BASE_URL}
ENV APP_API_BASE_URL=${APP_API_BASE_URL}

CMD node server.js


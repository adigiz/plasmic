# See https://hub.docker.com/_/node/
FROM node:24-alpine AS builder

RUN apk add --no-cache make~=4 bubblewrap~=0.11 python3 build-base bash git

# Build context is ./platform, so we need to copy and build dependencies first
# Copy sub directory (needed for wab build)
WORKDIR /app/sub
COPY sub/package.json sub/yarn.lock ./
RUN yarn install --frozen-lockfile && yarn cache clean
COPY sub/ .
RUN yarn build

# Copy and build react-web-bundle
WORKDIR /app/react-web-bundle
COPY react-web-bundle/package.json react-web-bundle/yarn.lock ./
RUN yarn install --frozen-lockfile && yarn cache clean
COPY react-web-bundle/ .
RUN yarn build

# Copy and build live-frame
WORKDIR /app/live-frame
COPY live-frame/package.json live-frame/yarn.lock ./
RUN yarn install --frozen-lockfile && yarn cache clean
COPY live-frame/ .
RUN yarn build

# Copy register-library first (needed by canvas-packages)
WORKDIR /app/wab
COPY wab/src/wab/shared/register-library ./src/wab/shared/register-library

# Copy and build canvas-packages (depends on register-library)
WORKDIR /app/canvas-packages
COPY canvas-packages/package.json canvas-packages/yarn.lock ./
# Copy files needed for prepare script (makePkgTypes.js runs after yarn install)
COPY canvas-packages/makePkgTypes.js ./
COPY canvas-packages/typedPkgsList.json ./
COPY canvas-packages/internal_pkgs ./internal_pkgs
RUN yarn install --frozen-lockfile && yarn cache clean
# Copy the rest of canvas-packages source files (including esbuild.js, hostlessList.json, etc.)
COPY canvas-packages/ .
RUN yarn build

# Now build wab (which depends on the above)
WORKDIR /app/wab
# Copy package files first
COPY wab/package.json wab/yarn.lock ./
COPY wab/patches/ ./patches/
# Copy the register-library local package first (needed for yarn install)
COPY wab/src/wab/shared/register-library ./src/wab/shared/register-library
# Now install dependencies (register-library will be found)
RUN yarn install --frozen-lockfile && yarn cache clean
# Copy the rest of wab source files
COPY wab/ .
RUN mkdir -p src/wab/gen && make
# Build the frontend for production (now that dependencies are built)
# Set optional environment variables for production build (required by build process, but can be placeholder values)
ENV AMPLITUDE_API_KEY="none"
ENV INTERCOM_APP_ID="none"
ENV POSTHOG_API_KEY="none"
ENV POSTHOG_HOST="none"
ENV POSTHOG_REVERSE_PROXY_HOST="none"
ENV SENTRY_DSN="none"
ENV SENTRY_ORG_ID="none"
ENV SENTRY_PROJECT_ID="none"
ENV STRIPE_PUBLISHABLE_KEY="none"
RUN yarn build
RUN rm -rf node_modules/

FROM node:24-alpine AS runner
# System setup
RUN apk add --no-cache make~=4 bubblewrap~=0.11 python3 build-base bash

# loader-bundle-env is expected to be a peer folder to wab, so we copy it there
WORKDIR /app/loader-bundle-env
COPY loader-bundle-env/package.json loader-bundle-env/yarn.lock ./
COPY loader-bundle-env/patches/ ./patches/
RUN yarn install --frozen-lockfile && yarn cache clean

# loader-html-hydrate is expected to be a peer folder to wab, so we copy it there
WORKDIR /app/loader-html-hydrate
COPY loader-html-hydrate/package.json loader-html-hydrate/yarn.lock ./
RUN yarn install --frozen-lockfile && yarn cache clean

WORKDIR /app/loader-bundle-env
COPY loader-bundle-env/ .
RUN ./yarn-install-internal-pkgs.sh
RUN ./yarn-build-internal-pkgs.sh

WORKDIR /app/loader-html-hydrate
COPY loader-html-hydrate/ .
RUN yarn build

# Expose main server port
EXPOSE 3004

# Switch workdir back to wab
WORKDIR /app/wab
COPY wab/package.json wab/yarn.lock ./
COPY wab/patches/ ./patches/
RUN yarn install --production --frozen-lockfile && yarn cache clean
COPY wab/ .
COPY --from=builder /app/wab/ .
# Copy the built frontend from builder
COPY --from=builder /app/wab/build ./build
COPY --from=builder /app/wab/public ./public

USER node

CMD ["node", "-r", "esbuild-register", "src/wab/server/main.ts"]

FROM node:22-alpine3.21 AS builder

# No need to change user or working directory for Kaniko
WORKDIR /directus

ARG TARGETPLATFORM

ENV NODE_OPTIONS=--max-old-space-size=8192

RUN \
  if [ "$TARGETPLATFORM" = 'linux/arm64' ]; then \
  apt-get update -qy && apt-get install -qy \
  python3 \
  build-essential \
  && ln -sf /usr/bin/python3 /usr/bin/python \
  ; fi

COPY package.json .
RUN corepack enable && corepack prepare

COPY pnpm-lock.yaml .
RUN pnpm fetch
COPY . .

RUN pnpm install --recursive --offline --frozen-lockfile
RUN pnpm update stdlib
RUN : \
  && npm_config_workspace_concurrency=1 pnpm run build \
  && pnpm --filter directus deploy --prod dist \
  && cd dist \
  && pnpm pack \
  && tar -zxvf *.tgz package/package.json \
  && mv package/package.json package.json \
  && rm -r *.tgz package \
  && mkdir -p database extensions uploads \
  ;

####################################################################################################
## Create Production Image

# Use a more compatible base image for Kaniko
FROM node:22-alpine3.21 AS runtime

# No need to change user or working directory for Kaniko
WORKDIR /directus

EXPOSE 8055

ENV \
  DB_CLIENT="sqlite3" \
  DB_FILENAME="/directus/database/database.sqlite" \
  EXTENSIONS_PATH="/directus/extensions" \
  STORAGE_LOCAL_ROOT="/directus/uploads" \
  NODE_ENV="production" \
  NPM_CONFIG_UPDATE_NOTIFIER="false"

# Copy files from the builder stage to the runtime stage
COPY --from=builder /directus/dist .

# You can optionally copy other necessary files here
# COPY cli.js cli.js
# COPY startup.sh startup.sh
# RUN chmod 777 startup.sh

# ENTRYPOINT [ "/directus/startup.sh" ]

CMD : \
  && node /directus/cli.js bootstrap \
  && node /directus/cli.js start \
  ;

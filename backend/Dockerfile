# syntax=docker/dockerfile:1.7-labs

# Stage 1: Build the application
FROM node:20 AS build

WORKDIR /usr/src/app
COPY . .

RUN --mount=type=cache,target=/root/.npm \
    npm ci && npm run build

# Stage 2: Create the final image without source files
FROM ghcr.io/ticketz-oss/node

ARG TICKETZ_REGISTRY_URL

WORKDIR /usr/src/app

# Instala o dos2unix para corrigir quebras de linha de scripts vindos do Windows
# A base ghcr.io/ticketz-oss/node provavelmente é Debian/Ubuntu, então usamos apt-get
RUN apt-get update && apt-get install -y dos2unix && rm -rf /var/lib/apt/lists/*

# Copia os artefatos da build
COPY --from=build /usr/src/app/dist/ ./dist/
COPY --from=build /usr/src/app/node_modules/ ./node_modules/
COPY --from=build /usr/src/app/package.json ./package.json
COPY --from=build /usr/src/app/scripts/ ./scripts/

# Converte os scripts para o formato Unix (LF) e garante permissão de execução
RUN dos2unix ./scripts/*.sh \
    && chmod +x ./scripts/*.sh

ENV NODE_ENV=production
ENV PORT=3000
ENV TICKETZ_REGISTRY_URL=${TICKETZ_REGISTRY_URL}

EXPOSE 3000

CMD dockerize -wait tcp://${DB_HOST}:5432 -timeout 60s \
  && npx sequelize db:migrate  --config dist/config/database.js --migrations-path dist/database/migrations \
  && ./scripts/load-retrieved.sh /retrieve; exit_code=$? \
  && if [ $exit_code -eq 1 ]; then npm run mark-seeds; exit 0; elif [ $exit_code -ge 100 ]; then exit 1; fi \
  && npx sequelize db:seed:all --config dist/config/database.js --seeders-path dist/database/seeds \
  && node dist/server.js

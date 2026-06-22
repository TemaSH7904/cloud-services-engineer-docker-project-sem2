# syntax=docker/dockerfile:1

# ===== Этап 1: сборка статики =====
FROM node:16-alpine AS builder
ARG VUE_APP_API_URL=/api
ENV VUE_APP_API_URL=$VUE_APP_API_URL
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# ===== Этап 2: отдача через nginx без root =====
# nginx-unprivileged работает под пользователем nginx (uid 101) и слушает 8080
FROM nginxinc/nginx-unprivileged:1.27-alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 8080

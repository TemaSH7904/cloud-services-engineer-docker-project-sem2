# syntax=docker/dockerfile:1

# ===== Этап 1: сборка бинарника =====
FROM golang:1.23-alpine AS builder

WORKDIR /app

# Сначала копируем только манифесты зависимостей.
# Пока go.mod и go.sum не меняются, этот слой берётся из кэша,
# и go mod download не запускается заново при каждой пересборке.
COPY go.mod go.sum ./
RUN go mod download

# Теперь исходный код
COPY . .

# Статическая сборка без CGO: бинарник не зависит от системных
# библиотек и запустится в минимальном образе.
# -ldflags="-s -w" убирает отладочную информацию и уменьшает размер.
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/server ./cmd/api

# ===== Этап 2: финальный образ =====
FROM alpine:3.20

# ca-certificates — на случай исходящих HTTPS-запросов из приложения.
RUN apk add --no-cache ca-certificates

# Непривилегированный пользователь: контейнер не должен работать
# под root (требование безопасности из чек-листа).
RUN addgroup -S app && adduser -S -G app app

WORKDIR /app

# Копируем из этапа сборки только готовый бинарник —
# ни компилятора, ни исходников, ни кэша в финальном образе нет.
COPY --from=builder /app/server /app/server

# Переключаемся на непривилегированного пользователя
USER app

# Бэкенд слушает порт 8081
EXPOSE 8081

ENTRYPOINT ["/app/server"]
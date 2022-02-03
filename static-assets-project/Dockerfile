FROM node:12-stretch As builder
WORKDIR /app
COPY . .
RUN npm ci && npm run build

FROM nginx:alpine AS runner
COPY --from=builder /app/build /usr/share/nginx/html

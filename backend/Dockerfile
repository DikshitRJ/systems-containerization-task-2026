FROM node:20-alpine

WORKDIR /app

RUN apk add --no-cache openssl

COPY . .

RUN npm ci
RUN npx prisma generate

EXPOSE 4000

CMD ["sh", "-c", "npx prisma migrate deploy && npm start"]


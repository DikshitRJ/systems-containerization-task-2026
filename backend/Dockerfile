FROM node:20-alpine

WORKDIR /app

RUN apk add --no-cache openssl

COPY --chown=node:node package*.json ./
COPY --chown=node:node prisma ./prisma/
RUN npm ci
RUN npx prisma generate

COPY --chown=node:node . .

USER node

EXPOSE 4000

CMD ["sh", "-c", "npx prisma migrate deploy && npm start"]


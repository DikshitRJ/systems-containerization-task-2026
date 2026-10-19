FROM node:20

WORKDIR /app

COPY . .
RUN npm install
ENV VITE_API_URL=""
RUN npm run build

EXPOSE 80
CMD ["npx", "serve", "-s", "dist", "-l", "80"]
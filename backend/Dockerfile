FROM node:20.11-alpine3.19

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

EXPOSE 5000

CMD ["node", "index.js"]

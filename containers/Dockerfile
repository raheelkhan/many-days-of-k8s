FROM node:20.2.0

WORKDIR /usr/src/app
COPY package*.json ./
COPY tsconfig.json ./
RUN npm install
COPY ./server.ts ./
RUN npm run build

CMD ["node", "server.js"]

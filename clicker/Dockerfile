FROM node:14

WORKDIR /usr/src/app
ENV PORT=80
EXPOSE 80

COPY package*.json ./
RUN npm ci

COPY . .

CMD [ "npm", "start" ]

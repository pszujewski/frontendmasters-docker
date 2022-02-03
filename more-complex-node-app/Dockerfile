FROM node

USER node

RUN mkdir /home/node/src

WORKDIR /home/node/src

COPY --chown=node:node . .

RUN npm ci

# npm ci is going to adhere to the package-lock. Otherwise, it's like npm install

CMD ["node", "index.js"]
FROM node:lts-slim

ENV NODE_ENV=production
WORKDIR /app

# Dependencies first: this layer is cached until package*.json changes
COPY package*.json ./
RUN npm install --omit=dev && npm cache clean --force

COPY . .

USER node
EXPOSE 3000
CMD ["node", "index.js"]

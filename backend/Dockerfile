FROM node:20-alpine
WORKDIR /app
COPY backend/package*.json ./
RUN npm ci --only=production && npm audit fix --force || true
COPY backend/ .
RUN mkdir -p uploads
EXPOSE 5000
CMD ["node", "server.js"]

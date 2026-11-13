FROM node:20-alpine

WORKDIR /app

# Copy dependency specifications
COPY package*.json ./

# Configure npm retry settings for reliable network downloads
RUN npm config set fetch-retries 5 && npm config set fetch-retry-mintimeout 10000 && npm config set fetch-retry-maxtimeout 60000

# Install production dependencies (includes tsx and typescript)
RUN npm ci --omit=dev --legacy-peer-deps

# Copy configuration and backend source code
COPY tsconfig.json ./
COPY backend/ ./backend/
COPY frontend/src/types/ ./frontend/src/types/

EXPOSE 5000

ENV NODE_ENV=production
ENV PORT=5000

CMD ["node", "./node_modules/tsx/dist/cli.mjs", "backend/index.ts"]


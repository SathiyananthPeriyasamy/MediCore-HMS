FROM node:20-alpine

WORKDIR /app

# Copy dependency specifications
COPY package*.json ./

# Install runtime dependencies and tsx for TypeScript execution
RUN npm ci --omit=dev --legacy-peer-deps && npm install tsx typescript --legacy-peer-deps

# Copy configuration and backend source code
COPY tsconfig.json ./
COPY backend/ ./backend/
COPY frontend/src/types/ ./frontend/src/types/

EXPOSE 5000

ENV NODE_ENV=production
ENV PORT=5000

CMD ["node", "./node_modules/tsx/dist/cli.mjs", "backend/index.ts"]


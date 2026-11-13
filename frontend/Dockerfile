# Stage 1: Build Frontend Assets
FROM node:20-alpine AS builder

WORKDIR /app

# Copy dependency specifications
COPY package*.json ./

# Configure npm retry settings for reliable network downloads
RUN npm config set fetch-retries 5 && npm config set fetch-retry-mintimeout 10000 && npm config set fetch-retry-maxtimeout 60000

# Install all dependencies required for building
RUN npm ci --legacy-peer-deps

# Copy configuration files and source code
COPY tsconfig.json vite.config.ts ./
COPY frontend/ ./frontend/
COPY backend/ ./backend/

# Build client assets into /app/dist
RUN npm run build

# Stage 2: Serve with Nginx Alpine
FROM nginx:alpine

# Copy built production assets
COPY --from=builder /app/dist /usr/share/nginx/html

# Copy custom Nginx reverse proxy configuration
COPY frontend/nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]


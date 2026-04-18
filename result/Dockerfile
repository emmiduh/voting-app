FROM node:24.5.0-slim

WORKDIR /usr/local/app

# Copy package requirements and install dependencies
COPY package*.json ./
RUN npm install && npm cache clean --force

# Copy the rest of the application code
COPY . .

# Expose the standard port the app runs on
ENV PORT=80
EXPOSE 80

# Start the application
CMD ["node", "server.js"]

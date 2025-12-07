#Stage 1: build React app
# Use the official Node.js 18 alpine image as a base image
FROM node:18-alpine AS build

# Set the working directory in the container
WORKDIR /usr/src/app

# Copy the package.json and package-lock.json files to the container
COPY package*.json ./

# Install the application's dependencies inside the container
RUN npm install

# Copy the rest of the application code to the container
COPY . .

RUN npm run build


# Stage 2: serve with Nginx
FROM nginx:alpine

COPY --from=build /usr/src/app/build /usr/share/nginx/html

EXPOSE 80

# Specify the command to run when the container starts
CMD [ "nginx", "-g", "demon off;" ]
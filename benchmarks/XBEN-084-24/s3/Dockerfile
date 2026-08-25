FROM node:14-alpine@sha256:434215b487a329c9e867202ff89e704d3a75e554822e07f3e0c0f9e606121b33

RUN apk --no-cache add curl
WORKDIR /app
COPY . .
RUN npm install 
EXPOSE 8333
CMD [ "npm", "start" ]

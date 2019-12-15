############################################
### APP BUILD STAGE
############################################
FROM node:carbon AS appbuild

# Create app directory
RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app

# Install app dependencies
COPY package.json /usr/src/app/
RUN npm install
COPY js /usr/src/app/js
COPY node_modules /usr/src/app/node_modules

############################################
### IMAGE BUILD STAGE
############################################
FROM node:carbon

# Create app directory
RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app

# Install app dependencies
COPY package.json /usr/src/app/
RUN npm install --production && npm cache clean --force

# Bundle app source
COPY --from=appbuild /usr/src/app/js /usr/src/app/js
COPY --from=appbuild /usr/src/app/node_modules /usr/src/app/node_modules
COPY config /usr/src/app/config

EXPOSE  3000

CMD [ "node", "dist" ]

FROM node:lts-alpine as build-stage
WORKDIR /app
RUN npm install -g @vue/cli todomvc todomvc-app-css && \
  vue create -d -f -n todomvc && \
  cd todomvc && \
  npm run build 

COPY index.html /app/todomvc/dist/index.html
COPY js /app/todomvc/dist/js
COPY node_modules /app/todomvc/dist/node_modules

FROM nginx:stable-alpine as production-stage
COPY --from=build-stage /app/todomvc/dist/ /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]

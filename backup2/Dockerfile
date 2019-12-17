FROM nginx:latest as web

WORKDIR /usr/share/nginx/html/
RUN rm -f index.html
EXPOSE 80

COPY ./index.html /usr/share/nginx/html/

ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
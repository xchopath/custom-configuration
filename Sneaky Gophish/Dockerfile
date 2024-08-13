FROM node:latest AS build-js

RUN npm install gulp gulp-cli -g

WORKDIR /build
COPY . .
RUN npm install --only=dev
RUN gulp

# Build Golang binary
FROM golang:1.15.2 AS build-golang

WORKDIR /go/src/github.com/gophish/gophish
COPY . .

RUN sed -i 's/X-Gophish-Contact/X-CSRF-Token/g' models/email_request_test.go
RUN sed -i 's/X-Gophish-Contact/X-CSRF-Token/g' models/maillog.go
RUN sed -i 's/X-Gophish-Contact/X-CSRF-Token/g' models/maillog_test.go
RUN sed -i 's/X-Gophish-Contact/X-CSRF-Token/g' models/email_request.go
RUN sed -i 's/X-Gophish-Signature/X-Token-Signature/g' webhook/webhook.go
RUN sed -i 's/const ServerName = "gophish"/const ServerName = "IGNORE"/' config/config.go
RUN sed -i 's/const RecipientParameter = "rid"/const RecipientParameter = "clientId"/g' models/campaign.go
RUN sed -i 's/ 7/ 40/g' models/result.go

RUN go get -v && go build -v

# Runtime container
FROM debian:stable-slim

RUN useradd -m -d /opt/gophish -s /bin/bash app

RUN apt-get update && \
	apt-get install --no-install-recommends -y jq libcap2-bin && \
	apt-get clean && \
	rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

WORKDIR /opt/gophish
COPY --from=build-golang /go/src/github.com/gophish/gophish/ ./
COPY --from=build-js /build/static/js/dist/ ./static/js/dist/
COPY --from=build-js /build/static/css/dist/ ./static/css/dist/
COPY --from=build-golang /go/src/github.com/gophish/gophish/config.json ./
RUN chown app. config.json

RUN sed -i 's/rid=/'"clientId"'=/g' static/js/src/app/campaign_results.js
RUN sed -i 's/rid=/'"clientId"'=/g' static/js/dist/app/campaign_results.min.js

RUN setcap 'cap_net_bind_service=+ep' /opt/gophish/gophish

USER app
RUN sed -i 's/127.0.0.1/0.0.0.0/g' config.json
RUN touch config.json.tmp

EXPOSE 3333 8080 8443 80

CMD ["./docker/run.sh"]

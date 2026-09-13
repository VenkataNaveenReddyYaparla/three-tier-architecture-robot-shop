FROM golang:1.24 AS builder

WORKDIR /go/src/app

COPY *.go .

RUN go mod init dispatch # This will create go.mod file
RUN go mod tidy  # downloads dependencies and creates go.sum
RUN CGO_ENABLED=0 go build -o dispatch . # CGO makes static binary no run time libraries

FROM alpine:latest

WORKDIR /app
COPY --from=builder /app/dispatch .

CMD ["./dispatch"]

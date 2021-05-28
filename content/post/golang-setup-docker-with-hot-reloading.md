+++
title = "Golang setup docker with hot reloading"
date = "2021-05-28"
description = "How to create a Dockerfile supporting hot reloading by CompileDaemon on Gin project example."
tags = ["golang", "gin"]
categories = ["golang"]
+++

Today I'm going to show you how to easily add hot reloading functionality to your 
Golang project. I will use a simple Gin project as an example and docker-compose.

To make code as short as possible I will use [Gin example](https://github.com/gin-gonic/gin#quick-start) 
with `GET ping` endpoint returning `{"message": "pong"}` as a response to request.

### **main.py**

{{< highlight golang "linenos=table" >}}
package main

import "github.com/gin-gonic/gin"

func main() {
	r := gin.Default()
	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
	r.Run() // listen and serve on 0.0.0.0:8080 (for windows "localhost:8080")
}
{{< /highlight >}}


### **docker/Dockerfile**

{{< highlight dockerfile "linenos=table,hl_lines=27" >}}
FROM golang:1.16.4-stretch as build

ENV PROJECT_DIR=/project/app \
    GO111MODULE=auto \
    CGO_ENABLED=0

## Install required packages
## - git to fetch golang dependencies
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        git \
    && rm -rf /var/lib/apt/lists/*

## Set the current working directory inside the container
WORKDIR $PROJECT_DIR

## Install dependencies from go.mod file and verify it also
COPY go.mod .
COPY go.sum .
RUN go mod download
RUN go mod verify

## Install additional dependencies
RUN go get github.com/githubnemo/CompileDaemon

## Run watcher to automatically rebuild app after any change in code
CMD CompileDaemon --build="go build src/main.go" --command="./main" --color
{{< /highlight >}}

At the end of the file, we are running [CompileDaemon](https://github.com/githubnemo/CompileDaemon)
written in pure golang who will do all the job for us. In `build` we can optionally put out own command to build project
by default is `go build`. `command` argument will run our binary created by build. Yes, is so easy :) 

### **docker-compose.yml**

{{< highlight docker "linenos=table" >}}
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: ./docker/Dockerfile
    volumes:
      - .:/project/app
    restart: on-failure
{{< /highlight >}}

## Testing time!

Run `docker-compose up --build` and check if I not lied to you :)

{{< highlight sh >}}
...
app_1  | Running build command!
app_1  | Build ok.
app_1  | Restarting the given command.
app_1  | stdout: [GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.
app_1  | stdout: 
app_1  | stdout: [GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
app_1  | stdout:  - using env:	export GIN_MODE=release
app_1  | stdout:  - using code:	gin.SetMode(gin.ReleaseMode)
app_1  | stdout: 
app_1  | stdout: [GIN-debug] GET    /ping                     --> main.main.func1 (3 handlers)
app_1  | stdout: [GIN-debug] Environment variable PORT is undefined. Using port :8080 by default
app_1  | stdout: [GIN-debug] Listening and serving HTTP on :8080
{{< /highlight >}}

Change `ping` endpoint response to something else and save your file.
You should see in your terminal new lines regarding rebuilding.

{{< highlight sh >}}
...
app_1  | Running build command!
app_1  | Build ok.
app_1  | Hard stopping the current process..
app_1  | Restarting the given command.
...
{{< /highlight >}}

Full source from this post you can find [here](https://github.com/bplociennik/bplociennik-examples/tree/master/golang-docker-hot-reloading), have fun!

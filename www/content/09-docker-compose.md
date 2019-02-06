---
title: Docker Compose
weight: 9
menu: true
---

With Docker CLI, we can launch invididual containers.
In this section, we'll cover a few examples on how to launch a fleet of containers using Docker Compose.

Docker Compose is a tool for defining and running multiple containers.
The containers are configured with [YAML](https://yaml.org/) formatted files,
and launched with the `docker-compose` command.

## Web app with a database

A common use-case for Docker Compose is to use for configuring and launching a quick environment for testing an application you are developing.
For example, if we were to develop a web application that stores data to a database,
then the Docker Compose setup would launch the app and the database the app uses.

Let's extend our web application to support storing data to a [Redis](https://redis.io/) database.
Copy and paste this updated source code to the Python app source file we created earlier.

```python
import os
from redis import Redis
from flask import Flask, request, jsonify
app = Flask(__name__)

db = None

@app.route('/')
def hello():
    return 'Hello World!\n'

@app.route('/data')
def alldata():
    return jsonify(db.hgetall('mydata'))

@app.route('/data/<id>', methods=['GET', 'POST'])
def data(id):
    if request.method == 'POST':
        db.hset('mydata', id, request.data)
        return 'OK\n'
    return db.hget('mydata', id)

if __name__ == '__main__':
    db = Redis(
        host=os.getenv('REDIS_HOST'),
        port=int(os.getenv('REDIS_PORT', '6379')),
        password=os.getenv('REDIS_PASSWORD', ''),
    )
    app.run(host='0.0.0.0', port=3000)
```

The application now has a `/data` endpoint,
which you can use to insert arbitrary data to a single Redis hash map named `mydata`.
The Redis connection details are configured using environment variables, which we will cover soon.

Also, we need to update the Dockerfile to include the Redis Python bindings from PIP.

```Dockerfile
FROM ubuntu:18.04
WORKDIR /opt/myapp
RUN apt-get update && \
    apt-get install -y python python-pip && \
    pip install --no-cache Flask redis && \
    apt-get remove -y python-pip && \
    apt-get autoremove -y && \
    rm -rf /var/lib/apt/lists/*
RUN groupadd -g 999 appuser && \
    useradd -r -u 999 -g appuser appuser
USER appuser:appuser
CMD ["python", "app.py"]
COPY app.py .
```

Now that we have our app source code updated,
we can create a Docker Compose file that runs both Redis and our app simultaneously in one script.
Create a file named `docker-compose.yml` next to the Dockerfile, and insert the following contents.

```yaml
version: '3'
services:
  redis:
    image: redis
  myapp:
    build: .
    ports:
      - "3000:3000"
    environment:
      REDIS_HOST: redis
```

The file defines two services: `redis` and `myapp`.
The Redis service uses [the Redis Docker image from Docker Hub](https://hub.docker.com/_/redis),
which is pre-configured to launch Redis on port 6379.
Our app will be built on-demand when we execute Docker Compose.

The app will expose port 3000 as usual, and it's configured via the environment variables to contact the Redis server listed above.
If we were to run the app container without Docker Compose,
we would supply the environment variables using the `-e` flag: `-e REDIS_HOST=redis`

Docker Compose automatically creates a dedicated bridge network for the setup.
We could also create custom networks in the Compose file by defining a `networks` section.

Let's launch the containers with `docker-compose`.

    ~/myapp $ docker-compose up -d

The fleet is launched in the background because we used the `-d` flag.
If we were to leave it out, the fleet would start in the foreground.

We can test that the setup works by sending some data using `curl`.

    $ curl -X POST -H 'Content-Type: text/plain' -d myvalue http://localhost:3000/data/mykey
    OK
    $ curl http://localhost:3000/data/mykey
    myvalue
    $ curl http://localhost:3000/data
    {"mykey":"myvalue"}

Great! We can now tear down the setup.

    ~/myapp $ docker-compose down

## Integration tests

Now that we have a local development environment for our app,
we can extend the setup to run some integration tests for us.

Let's create a shell script that acts as our integration test.
Write the shell script to file `integration_test.sh` next to the `docker-compose.yml` file we just created.

```bash
#!/usr/bin/env sh
set -e # Fail script on command error

APP_URL="http://$APP_HOST:$APP_PORT"

# Poll server until it's ready
while true; do
    if curl -sfL "$APP_URL" >/dev/null; then
        break
    fi
done

# Test
EXPECTED=mytestvalue
curl -sfL -X POST -H 'Content-Type: text/plain' -d "$EXPECTED" "$APP_URL/data/mytestkey"
ACTUAL=$(curl -sfL "$APP_URL/data/mytestkey")

# Check result
if [ "$ACTUAL" = "$EXPECTED" ]; then
    echo "Test success!"
else
    echo "Test failed! $ACTUAL != $EXPECTED"
    exit 1
fi
```

Make sure the script is executable:

    ~/myapp $ chmod +x integration_test.sh

In the script above, we insert a value using the app `/data` endpoint, read it back, and verify that what we read is what we wrote.
If the test fails, the script exits with an error code. Otherwise, the script will exit normally.
Normally, you'd most likely want to use a proper test framework, but this should be enough for this exercise.

Before the test is executed, the script polls the server until it can contact it.
This is done to avoid race conditions because Docker Compose launches all the containers simultaneously.

Now we need to have this script executed as part of the Docker Compose setup.
Instead of extending the existing `docker-compose.yml` file,
we'll create a another similar file that contains only the test.
We'll later merge the two compose files together.
Create a file named `docker-compose-test.yml` next to the `docker-compose.yml` file.

```yaml
version: '3'
services:
  mytest:
    image: appropriate/curl
    entrypoint:
      - /integration_test.sh
    command: []
    environment:
      APP_HOST: myapp
      APP_PORT: 3000
    volumes:
      - ./integration_test.sh:/integration_test.sh
```

For running the test, we'll use the `appropriate/curl` image from the earlier exercise.
The test script will be brought in as a volume, and the configuration details via environment variables.

Since the `appropriate/curl` image uses `curl` as the entrypoint program,
we cannot launch the script by overriding the command.
Instead, we'll need to override the entrypoint with our test script.

When we run Docker Compose, we can use the `-f` option multiple times to provide custom Docker Compose scripts.
This will combine the scripts to one script before it's run.

We can use the flag `--abort-on-container-exit` to cause the entire setup to tear down automatically when any of the containers exit.
This way the test script can automatically cause the setup to be torn down when it exits.
Moreover, the exit code from the first container to exit will be the exit code for `docker-compose`,
which we can use to determine whether the test was successful or not.

Let's run the test Docker Compose script in combination with the existing Docker Compose script.

    $ docker-compose -f docker-compose.yml -f docker-compose-test.yml up --abort-on-container-exit
    $ echo $?
    0

If the test succeeded, the exit code should be `0`.
Try editing the integration test script to cause the test to fail, and re-run the Docker Compose command.

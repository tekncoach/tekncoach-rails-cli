# Rails CLI Docker Image 

Setting up Rails for the first time with all the dependencies necessary can be daunting for beginners. 
[Docked Rails CLI](https://github.com/rails/docked) uses a Docker image to make it much easier, requiring only Docker to be installed.
But this image doesn't include the necessary headers required to start a Rails app with `Postgress`

Generate a Docker image for Ruby On Rails based on the last version of Ruby
and including Postgres headers and some extra bins needed in real projects

## Docker setup on a new Mac

Install [Docker for Mac](https://docs.docker.com/desktop/install/mac-install/) or [OrbStack](https://orbstack.dev)

```sh
brew install orbstack
```

## Build the `tekncoach/rails/cli` Docker image

1. Edit the `RUBY_VERSION` as needed in the `Dockerfile.rails` 
2. Run the command to generate a specific image with all the needed dependencies specified and load it into `Docker`.

```sh
cd tekncoach-rails-cli
./bin/load
```

Then copy'n'paste into your terminal:

```sh
docker volume create ruby-bundle-cache
alias docked='docker run --rm -it -v ${PWD}:/rails -v ruby-bundle-cache:/bundle tekncoach/rails/cli'
alias dockeds='docker run --rm -it -v ${PWD}:/rails -v ruby-bundle-cache:/bundle -p 3000:3000 tekncoach/rails/cli'
```

You can now use it by referencing it in your `docker-compose.yml` config files or in your aliases on the command line

Then create your Rails app:
Note that you need a running PostgreSQL server running to be able to use and run the rails app on a postgresql database. 
You might need to use a `docker-compose.yml` config file for this.

```bash
docked rails new weblog # --database=postgresql
cd weblog
docked rails generate scaffold post title:string body:text
docked rails db:migrate
# Run the "rails server" with the command passing the local port to Docker. It allow running the "docked" command for others actions while a terminal with only the server is running with the allocated port 3000.
dockeds rails server
```

That's it! Your Rails app is running on `http://0.0.0.0:3000/posts`.

## Add Aliases in the Terminal

Open your aliases for your Terminal. I'm using Zsh here.

```bash
# With VSCode
code ~/.zsh_aliases
```

And add the following lines:

```bash
# Rails Docked alias 2023
# https://github.com/rails/docked
alias docked='docker run --rm -it -v ${PWD}:/rails -v ruby-bundle-cache:/bundle tekncoach/rails/cli'
alias dockeds='docker run --rm -it -v ${PWD}:/rails -v ruby-bundle-cache:/bundle -p 3000:3000 tekncoach/rails/cli'
alias rails='docked rails'
alias rails-dev='docked bin/dev'
alias bundle='docked bundle'
alias yarn='docked yarn'
alias rake='docked rake'
alias gem='docked gem'
alias rails-console='docked rails console'
```

## Test it with a new Rails app

Then create your Rails app:

```bash
docked rails new weblog
cd weblog
docked rails generate scaffold post title:string body:text
docked rails db:migrate
docked rails server
```

That's it! Your Rails app is running on `http://localhost:3000/posts`.

## Use it in your `docker-compose.yml` config file

```yaml
# docker-compose.yml
version: '3.8'

x-rails: &rails
build:
  context: "."
  args:
    UID: "${UID:-1000}"
    GID: "${GID:-${UID:-1000}}"
    RUBY_VERSION: '3.2.2'
  dockerfile: Dockerfile.dev
image: tekncoach/rails/cli
env_file: .env
tmpfs:
  - /tmp
# Keeps the stdin open, so we can attach to our app container's process and
# do stuff such as `byebug` or `binding.pry`
stdin_open: true
# Allows us to send signals (CTRL+C, CTRL+P + CTRL+Q) into the container
tty: true
volumes:
  - .:/rails:cached
  - ruby-bundle-cache:/bundle
depends_on:
  - db
  - redis

services:
web:
  <<: *rails
  command: bash -c "rm -f tmp/pids/server.pid && bin/dev web=1,all=0"
  ports:
    - '3000:3000'
  # container_name: tekncoach_rails

assets:
  <<: *rails
  command: bash -c "bin/dev web=0,all=1"
  depends_on:
    - web

redis:
  image: "redis:7-alpine"
  ports:
    - 6379
  volumes:
    - ./tmp/redis_data:/var/lib/redis/data

db:
  image: postgres
  restart: always
  environment:
    - POSTGRES_HOST_AUTH_METHOD=trust
  ports:
    - 5432:5432
  volumes:
    - ./tmp/postgres_data:/var/lib/postgresql/data

volumes:
ruby-bundle-cache:
  external: true
```

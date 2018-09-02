# wait-for
> Waits for a host to be either up or down and executes a command.

`wait-for` is a `sh` script that waits on the availability or unavailability of a host and then executes a command.

This script is intended to be used as part of docker-compose to synchronize the spin-up of interdependent services. For example, running migrations after a database is up, and then running the application once migrations are finished. Only has dependencies on `sh`, `nc` and `ping`, so it can be used in [Alpine][alpine] images.

It is heavily inspired by [wait-for-it][wait-for-it] and [wait-for][wait-for], adding some extra features like waiting for a container to be down.

## Usage

```
wait-for [options] [-- command args]
  -?, --help                    Display help.
  -h, --host HOST               Set the host that will be checked.
                                In wait-for mode, the target must be a host and port separated by a colon (:).
                                In wait-for-exit mode, the target can be anything accepted by ping.
  -t, --timeout TIMEOUT         Set the timeout for the check to fail.
                                (default: 3600 seconds)
  -m, --mode MODE             	Specifiy the mode for this script.
                                "wait-for-up" waits until the given host is up, using netcat.
                                "wait-for-down" waits until the given host is down, using ping.
                                (default: wait-for-up)
  -q, --quiet                   Do not display any output.
  -v, --verbose                 Display trace information.

  -- command args               Execute command with args after the probe finishes
```

## Examples
### On the command line
Test if google.com:80 is available, and echo a message.
```bash
$ ./wait-for -h google.com:80 -- echo "Google is up"
```

Test if db-migrations (an example container host) is down, and echo a message.
```bash
$ ./wait-for -h db-migrations -m wait-for-down -- echo "DB migrations are finished"
```
### As part of docker-compose
This script really shines as part of a `docker-compose.yml` files, where it can be used to orchestrate the spinning up of different containers. In the following example, an ephemeral container used for migrations waits until the db is up and running to start and the app waits until that container is finished, so the database is ready with all the needed data.
```
version: '3.4'

services:
  app:
    image: your-app-image
    entrypoint: ["/compose/wait-for", "--host=db-migrations", "--mode=wait-for-down", "--", "your-normal-entrypoint", "with-args"]
    volumes:
      - ./scripts:/compose
    depends_on:
      - db
      - db-migrations

  db:
    image: postgres:10-alpine
    ports:
      - "5432:5432"

  db-migrations:
    image: your-migrations-image
    entrypoint: ["/compose/wait-for", "--host=db:5432", "--mode=wait-for-up", "--", "run", "migrations", "up"]
    volumes:
      - ./scripts:/compose
    depends_on:
      - db
```

In this example a volume is mounted to provide access to the script. This allows its usage without modifying the source `Dockerfile`. It could also be embedded into the image with a `COPY` instruction, and then used in the `ENTRYPOINT` in pretty much the same way as in this example.

It is important to remember that when using *exec* form (which is the recommended form) the executable is called directly and shell processing does not happen. So, if the container should have access to environment variables, either use *shell* form on the `ENTRYPOINT`, or add the shell interpreter in the first place:

```
entrypoint: ["sh", "/compose/wait-for", "--host=db:5432", "--mode=wait-for-up", "--", "run", "migrations", "up"]
```

## Release History

* 0.1.0
    * Initial release.

## Contributing

1. Fork it (<https://github.com/carsanlop/wait-for/fork>)
2. Create your feature branch (`git checkout -b feature/fooBar`)
3. Commit your changes (`git commit -am 'Add some fooBar'`)
4. Push to the branch (`git push origin feature/fooBar`)
5. Create a new Pull Request

[wait-for-it]: https://github.com/vishnubob/wait-for-it
[wait-for]: https://github.com/eficode/wait-for
[alpine]: https://alpinelinux.org/
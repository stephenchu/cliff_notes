# Docker

## [Dockerfile Best Practices](https://docs.docker.com/articles/dockerfile_best-practices) (as of v1.5)

### `.dockerignore`

Use `.dockerignore` to exclude files like `.git` from being included in the image to reduce its final size.

### General

1. Avoid installing "nice-to-haves" into your image, such as a `vim` editor in a database image.
1. Run only a single process in a container.

### Build Caching
1. Every instruction in a `Dockerfile` can be a caching point (i.e. a layer) when we run `docker build`. This reduces time to rebuild an image from scratch each time `docker build` is run.
	* Use `--no-cache` if this is undesired behavior.
	* Only `ADD` and `COPY` will Docker perform a checksum on the files involved with those inside of its build cache. Instructions like `RUN apt-get -y update` will not trigger a files content checksum match to look for a prevously built cache.
	* Tip: While in image development, run multiple instructions to leverage on caching. When everything is ready, group as many instructions into one layer/instruction so save space and kittens [\[source\]](https://medium.com/@vaceletm/docker-layers-cost-b28cb13cb627).

### `Dockerfile` Instructions

#### `FROM`
* Use official repositories.

#### `RUN`
* Break up long `RUN` statements into multi-line.
* When you are running `apt-get`:
	* Do not do `RUN apt-get update` on a single line by itself. Otherwise subsequent `apt-get install` will cause caching issues.
	* Instead, do `RUN apt-get update && apt-get install -y package-xyz` in a single line.
	* Do not do `RUN apt-get upgrade` or `dist-upgrade`, since many of the essential packages will fail to upgrade in a unpriviledged container. Contact maintainer if base package is out of date.

#### `CMD`
* Should be used to run the software inside of the image. e.g. `RUN ["apache2", "-DFOREGROUND"]`.
* Should be given an interactive shell. e.g. `CMD ["python"]`. This allows someone to drop into a usable shell with `docker run -it python`.
* Avoid using `CMD ["param", "param"]` in conjunction with `ENTRYPOINT`, unless you know what you are doing.


#### `EXPOSE`
* Exposes ports on which a container will listen for connections. For external access, `docker run` with `-P` or `-p ...`.

#### `ENV`
* Can use it to make software in container easier to run by appending to `PATH`. e.g. `ENV PATH /usr/local/nginx/bin:$PATH` ensures `CMD ["nginx"]` will just work.
* Can use it to supply environment variables specific to the software inside of the container to run. e.g. `PGDATA`.
* Can use it to set commonly used version numbers so version bumps are easier to maintain.

      ENV PG_MAJOR 9.3
      ENV PG_VERSION 9.3.4
      RUN curl -SL http://example.com/postgres-$PG_VERSION.tar.xz | tar -xJC /usr/src/postgress && ...
      ENV PATH /usr/local/postgres-$PG_MAJOR/bin:$PATH
      
* Use `docker run -t ... env` to list environment variables.

#### `ADD` or `COPY`
* Generally, `COPY` is preferred.
* Best use of `ADD` is local tar file auto-extraction into the image. e.g. `ADD rootfs.tar.xz /`.
* Use multiple `COPY` and make use of what you are copying immediately after to ensure each step's build cache is only invalidated at the last minute.
* Do not use `ADD` to fetch remote packages. Use `curl` or `wget` instead. i.e.

  Insead of:
  
      ADD http://example.com/big.tar.xz /usr/src/things/
      RUN tar -xJf /usr/src/things/big.tar.xz -C /usr/src/things
      RUN make -C /usr/src/things all
      
  Do:
  
      RUN mkdir -p /usr/src/things && curl -SL http://example.com/big.tar.gz \
        | tar -xJC /usr/src/things && make -C /usr/src/things all
        
#### `ENTRYPOINT`
* Useful because the docker image name can double as a reference to the binary. e.g.

      ENTRYPOINT ["s3cmd"]
      CMD ["--help"]
    
      $ docker run s3cmd
      $ docker run s3cmd ls s3://mybucket
    
* The Official Postgres Image script:

      #!/bin/bash
      set -e

      if [ "$1" = 'postgres' ]; then
          chown -R postgres "$PGDATA"

          if [ -z "$(ls -A "$PGDATA")" ]; then
              gosu postgres initdb
          fi

          exec gosu postgres "$@"
      fi

      exec "$@"

	Note: This script uses the `exec` Bash command so that the final running application becomes the container's PID 1. This allows the application to receive any Unix signals sent to the container.
	
	The script is copied into the docker container:
	
      COPY ./docker-entrypoint.sh /
      ENTRYPOINT ["/docker-entrypoint.sh"]
      
#### `VOLUME`
* Used to expose any database storage area, configuration storage, or files/folders created by the docker container. Used for any mutable and/or user-serviceable parts of your image.

#### `USER`
* If a service can run without privileges, use USER to change to a non-root user.
	* Start by creating the group and user: `RUN groupadd -r postgres && useradd -r -g postgres postgres`
		* Note: UID/GID are non-deterministic. Assign explicit UID/GID if necessary.
* Avoid installing/using `sudo` since it has unpredictable TTY and signal-forwarding behavior (see [explanation](https://github.com/tianon/gosu)).
	* If you need functionality similar as `sudo` (e.g. init the daemon as root, but run as non-root), use [`gosu`](https://github.com/tianon/gosu).
* Lastly, to reduce layers and complexity, avoid switching USER back and forth frequently.

#### `WORKDIR`
* For clarity, always avoid `RUN cd … && do-something`.

#### `ONBUILD`
* Useful only for images going to be built `FROM` a given image. For example, you use `ONBUILD` for a language stack image that builds arbitrary user software written in that language within the `Dockerfile`. See [Ruby's `ONBUILD` variants](https://github.com/docker-library/ruby/blob/master/2.1/onbuild/Dockerfile).
* Images built from `ONBUILD` should get a separate tag, e.g. `ruby:1.9-onbuild`, `ruby:2.0-onbuild`.
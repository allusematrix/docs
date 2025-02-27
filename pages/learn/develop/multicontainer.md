---
title: Multiple containers
excerpt: A guide to running multiple containers with Docker Compose and {{ $names.company.lower }}
---

# Multiple containers

As your apps grow more complex, you may find significant benefit in running some services in separate containers. Splitting your app into multiple containers allows you to better isolate and maintain key services, providing a more modular and secure approach to fleet management. Each service can be packaged with the operating environment and tools it specifically needs to run, and each service can be limited to the minimum system resources necessary to perform its task. The benefits of multicontainer fleets compound as the complexity of the app grows. With multicontainer, each service is updated independently. Hence, larger apps can be developed and maintained by separate teams, each free to work in a way that best supports their service.

__Note:__ For additional information on working with multiple containers with {{$names.company.lower}} see the [services masterclass][services-masterclass].

This guide will cover the considerations you need to take into account when running multiple containers, including `docker-compose.yml` configuration and some important {{ $names.company.lower }} specific settings.

## docker-compose.yml file

The multicontainer functionality provided by {{ $names.company.lower }} is built around the **[Docker Compose][docker-compose]** file format. The {{ $names.company.lower }} device supervisor implements a subset of the [Compose v2.1 feature set][compose-features]. You can find a full list of supported and known unsupported features in our [device supervisor reference docs][compose-support].

At the root of your multicontainer release, you'll use a `docker-compose.yml` file to specify the configuration of your containers. The `docker-compose.yml` defines the services you'll be building, as well as how the services interact with each other and the host OS.

Here's an example `docker-compose.yml` for a [simple multicontainer release][simple-app], composed of a static site server, a websocket server, and a proxy:

```yaml
version: '2'
services:
  frontend:
    build: ./frontend
    restart: always
    expose:
      - "80"
  proxy:
    build: ./haproxy
    depends_on:
      - frontend
      - data
    ports:
      - "80:80"
  data:
    build: ./data
    expose:
      - "8080"
```

Each service can either be built from a directory containing a `Dockerfile`, as shown here, or can use a **Docker** image that has already been built, by replacing `build:` with `image:`. If your containers need to be started in a specific order, make sure to use the `depends_on:` [setting][depends-on].

__Note:__ Note that `depends_on` only controls the startup order and will not restart services if a dependency restarts. Also if a service is expected to stop after performing some actions, do not include it as a dependency or the service that depends on it may not start.

Unlike single container fleets, multicontainer fleets do not run containers in privileged mode by default. If you want to make use of hardware, you will either have to set some services to privileged, using `privileged: true`, or use the `cap_add` and `devices` settings to map in the correct hardware access to the container. 

Also, in a balena device, unlike with regular docker compose, containers restart by default if their process crashes or the device is restarted. This behavior can be changed by setting the restart policy to one of the [values supported by the engine](https://docs.docker.com/compose/compose-file/compose-file-v2/#restart) (the default is `restart: always`).

As an example, here the `gpio` service is set up to use i2c and serial uart sensors and a restart policy of `on-failure`. 

```Dockerfile
gpio:
    build: ./gpio
    devices:
      - "/dev/i2c-1:/dev/i2c-1"
      - "/dev/mem:/dev/mem"
      - "/dev/ttyACM0:/dev/ttyACM0"
    cap_add:
      - SYS_RAWIO
    restart: on-failure
```

## {{ $names.company.upper }} settings

There are a few settings and considerations specific to {{ $names.company.lower }} that need to be taken into account when building multicontainer fleets.

### Network mode

Setting `network_mode` to `host` allows the container to share the same network namespace as the host OS. When this is set, any ports exposed on the container will be exposed locally on the device. This is necessary for features such as bluetooth.

### Named volumes

With multicontainer fleets, {{ $names.company.lower }} supports the use of named volumes, a feature that expands on the [persistent storage][persistent-storage] functionality used by older versions of {{ $names.os.lower }}. Named volumes can be given arbitrary names and can be linked to a directory in one or more containers. As long as every release of the fleet includes a `docker-compose.yml` and the volume name does not change, the data in the volume will persist across updates.

Use the `volumes` field of the service to link a directory in your container to your named volume. The named volume should also be specified at the top level of the `docker-compose.yml`:

```yaml
version: '2'
volumes:
    resin-data:
services:
    example:
        build: ./example
        volumes:
            - 'resin-data:/data'
```

For devices upgraded from older versions of {{ $names.os.lower }} to v2.12.0 or higher, a link will automatically be created from the `/data` directory of the container to the `resin-data` named volume (similar to above). This ensures fleet behavior will remain consistent across host OS versions. One notable difference is that accessing this data via the host OS is done at `/var/lib/docker/volumes/<FLEET ID>_resin-data/_data`, rather than the `/mnt/data/resin-data/<FLEET ID>` location used with earlier host OS versions.

### Core dumps for containers

{{ $names.os.upper }} v2.113.31 and later, by default, will not generate core dump files when a container crashes. This prevents a buggy container that is crash looping to fill up all available storage space on a device.

[Core dumps][core-dump-link] are files with the contents of the memory used by a process at the time it crashes. They are a relatively advanced troubleshooting tool, particularly useful for low-level debugging of programs written in languages that compile to native code. For example, a tool like the `gdb` debugger can read a core dump and provide a stack trace showing the sequence of functions calls that led to the crash.

If you need core dumps, you can easily enable them by editing your `docker-compose.yml` and setting `ulimits.core: -1` for the desired services. For example:

```yaml
version: '2'
services:
    example:
        ulimits:
            core: -1
```

### Labels

In addition to the settings above, there are some {{ $names.company.lower }} specific labels that can be defined in the `docker-compose.yml` file. These provide access to certain bind mounts and environment variables without requiring you to run the container as privileged:

{{> "general/labels" }}

[docker-compose]:https://docs.docker.com/compose/overview/
[simple-app]:{{ $links.githubLabs }}/multicontainer-getting-started
[compose-features]:https://docs.docker.com/compose/compose-file/compose-file-v2/
[compose-support]:/reference/supervisor/docker-compose
[depends-on]:https://docs.docker.com/compose/compose-file/compose-file-v2/#depends_on
[persistent-storage]:/learn/develop/runtime/#persistent-storage
[init-system]:/learn/develop/runtime/#init-system
[app-types]:/learn/manage/app-types
[services-masterclass]:/learn/more/masterclasses/services-masterclass/
[core-dump-link]:https://en.wikipedia.org/wiki/Core_dump

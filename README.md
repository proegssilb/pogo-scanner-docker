# pogo-scanner-docker
[![Build Status](https://travis-ci.org/proegssilb/pogo-scanner-docker.svg?branch=master)](https://travis-ci.org/proegssilb/pogo-scanner-docker)

Some docker images to make setting up RocketMap "easier".

**NOTE:** Software should be used for educational purposes only, and is not
guaranteed to work for your particular use case. No support/help is available,
but pull requests are always welcome.

# What does a scalable RocketMap setup look like?
There are several ways to do it. However, not all approaches scale up equally. I
will only describe one way of doing things. First, the major pieces:

  1. One (or more) computers to run software. They need to be online at all hours,
  and not reboot automatically (say, for updates).
  1. Docker puts complex software in a box to make setup/deployment easier. It can
  also help with orchestrating cross-machine deployments.
  2. One (or more) databases. For modest setups, you probably only need one database.
  Needs to be MySql or MariaDB (A MySql-clone).
  3. At least one PokeAlarm instance, to forward alerts from RocketMap to other
  services.
  4. An instance of RocketMap with `-os`. This will setup a dedicated front-end
  instance, allowing better isolation/management (for example, being able to
  setup dedicated memory for other containers, while the front-end takes what
  is left).
  5. Several instances of RocketMap with `-ns`, one for each area you need to scan.
  Instead of setting up one big area, you should setup several smaller areas to
  tailor coverage and share the workload across CPU cores or computers.

The scanning instances of RocketMap interact with Pokemon Go servers to find pokemon.
Then, it updates the database with information about what it finds, and current
status, and sends webhooks to PokeAlarm. Using filters and alarms configured,
PokeAlarm determines what notices to send where. When someone wants to look at
a graphical map, they load a web page from the front-end instance(s) of RocketMap,
which interacts with the database.

That whole setup is software, and needs a computer to run on. Docker makes it so
you don't have to worry about installing RocketMap, its dependencies, making sure
whatever it is built properly, or whatnot. Just get Docker working, do some light
configuring, and run.

# Setup & Configuration

If you are not comfortable running command-line, TURN BACK. This tutorial
assumes you will be using a terminal the entire time. There are graphical tools
for working with Docker, but I don't know them. Feel free to submit a pull
request if you can fill in the gaps.

## Docker

First, if you haven't [installed docker](https://www.docker.com/community-edition)
already, do that. You're going to have to learn a bit about docker in order to
get everything working, unfortunately. For the average person, I recommend the
following topics:

  - Basic invocation of `docker` and `docker-compose`
  - Docker mounts, bind-mounts, and "single-file" mounts
  - Port mapping
  - Environment variables
  - `docker-compose up`, `docker-compose down`, and what "rebuilding containers"
  means.

You're going to want to pick a place on your computer where you can stick a bunch
of config files for RocketMap, PokeAlarm, and MariaDB. All of these files need
to be able to be mounted into a Docker Container. This means different things to
different platforms, due to platform-specific implementation details:

  - **Linux**: It needs to be somwhere you can refer to. Permissions are relevant.
  - **Windows 10 Home**: All files need to be in My Documents.
  - **Other Platforms**: I don't know. Ideally, same as Linux, but who knows.

TODO: Recommend some specific readings. Preferably tutorial-like.

Once you've done your research, copy the `docker-compose.example.yaml` file to
`docker-compose.yaml`. We'll be working on this file throughout the next few
sections. The example assumes you're on linux, and that all your data is in the
same directory as your `docker-compose.yaml` file, and that you'll run
`docker-compose` from that directory. You should tweak the filenames listed to
match your reality.

## MariaDB

Like most software, RocketMap has a fair bit of data to manage. It chooses to do
so via a SQL Database. In our case, this SQL database is MariaDB, a clone of
MySQL. For the most part, you should only have to worry about setting credentials.

If you are severely high-scale (if you have to ask, you aren't), you probably
already know enough to go research databases in containers yourself, and debate
whether following this section is a good idea. Do not shortcut yourself on this
process.

Create an Env File wherever you're keeping your config files for RocketMap and
PokeAlarm. In the example compose file, this is the file referred to by the
`env_file` key. It is used primarily for things that really shouldn't be anywhere
near github. In the env file, put entries for the following variables:

  - MYSQL_DATABASE
  - MYSQL_USER
  - MYSQL_PASSWORD
  - MYSQL_RANDOM_ROOT_PASSWORD=yes
  - MYSQL_ROOT_PASSWORD

We'll be adding more to this env file later, but that's it for now. I strongly
advise setting `MYSQL_RANDOM_ROOT_PASSWORD` to `yes`. Everything else is up to
whoever is setting up the RocketMap instances.

MYSQL_DATABASE controls which DB in MariaDB rocketmap should use. This is required
by the underlying MariaDB image to function properly. `MYSQL_USER` and
`MYSQL_PASSWORD` provide the credentials used for RocketMap to connect to its
database. These are for ensuring RocketMap has limited access to the rest of the
database (and any possible security holes). Don't set MYSQL_USER to root.
MYSQL_ROOT_PASSWORD is simply there for completeness, but to be on the safe side,
you should set it to a random string that you won't remember.

Designate a directory for MariaDB to store its data files in. You'll need to
mount that directory into Docker (in the compose file, whichever `service` has
`image: mariadb`, `volumes` key).

In the docker-compose file, take a look at whichever key under `services` has
`image: mariadb`. If you haven't already, adjust the keys `env_file` and `volumes`, as
appropriate. Leave the keys `networks` and `image` unchanged (unless you know
what you're doing...).

## RocketMap

OK, now we start to get interesting. First, let's set some more environment
variables in the same env file we used with MariaDB:

  - POGOMAP_DB_NAME
  - POGOMAP_DB_USER
  - POGOMAP_DB_PASS
  - POGOMAP_DB_HOST
  - POGOMAP_STATUS_PAGE_PASSWORD

Most of these need to be copy-paste from MariaDB. You should set POGOMAP_DB_PASS
to be the same value as MYSQL_PASSWORD, POGOMAP_DB_NAME to MYSQL_DATABASE,
and POGOMAP_DB_USER to MYSQL_USER. POGOMAP_STATUS_PAGE_PASSWORD can be whatever
you want. But, if you do set it, and want to actually access RocketMap's status
page, you need to remember what you set POGOMAP_STATUS_PAGE_PASSWORD to.

POGOMAP_DB_HOST is more complicated. In the example docker-compose file, I set
the MariaDB container to the name `rmdb`. This comes from the key under
`services` that contains all the config info associated with `image: mariadb`
In the example compose file, this key is `rmdb`. So, `POGOMAP_DB_HOST` would be
set to `rmdb`.

Next, copy `config.ini.example` into a directory you can mount into a docker
container, and modify it as needed for your use. At a minimum, you should change
the Google Maps and captcha keys, and webhook settings. See
[the RocketMap docs](https://rocketmap.readthedocs.io/en/develop/first-run/commandline.html)
for more details. All command-line flags should have an equivalent config file
option. Then, make sure you have a CSV file of accounts for each worker, stored
in the same place as the config file you modified earlier.

Lastly, in the compose file, there's a number of variables under the
`environment` key for each RocketMap container. RM_NAME is a unique name for
each RocketMap instance. RM_LOC is a GPS location (lattitude-longitude) for that
instance to start at. RM_ACCOUNTS is where  RocketMap can find its list of
accounts inside the container. Lastly, RM_MODE is `-ns` for scanner RocketMaps,
and `-os` for RocketMaps that run the UI. Each of  the environment variables
given needs to be set per-container. Each scanner instance should have a unique
RM_NAME, RM_LOC, and RM_ACCOUNTS. The front-end instances (with RM_MODE set to
`-os`) should have a unique RM_NAME, and RM_LOC can be wherever is convenient
for users. Edit the variables for each RocketMap instance in the compose file as
appropriate for your setup.

## PokeAlarm

PokeAlarm can be configured to use multiple "managers". This is required if you
need to, for example, send alerts to multiple channels in Discord, or multiple
accounts on Twitter. For each manager, you need an `alerts.json` and a
`filters.json`. If you don't need geofencing, the command line arguments to do this
might look like `-m2 -f filter1.json -a alert1.json -f filter2.json -a alert.json`.
Each manager's config files are listed together, and then the next manager's files,
and so on. For details about the exact available options in PokeAlarm, consult
[the PokeAlarm documentation](https://github.com/RocketMap/PokeAlarm/wiki). There's
a lot of extensive documentation, and I really don't have space here to describe
half of it. Once you've got your config files for PokeAlarm, make sure they are
in a directory you can mount into a Docker container.

All the magic for getting PokeAlarm to work is in the command-line arguments we
pass to it. Look for the service with `image: proegssilb/pokealarm`. Set the
`volumes` to match your setup. The `command` key is where we set the
command-line arguments. This setup is a bit unusual for docker (and maybe
unintuitive), but not unprecedented. The example sets up a single manager with
just a filter and an alarm file. Adjust as you see fit, following the sample
pattern of using a list of quoted items, one for each logical chunk of your
arguments.

# Testing

After the compose file is edited, you should be able to run `docker-compose up`
from the same directory as your docker-compose file and have RocketMap start up.
Look out for lines that talk about a container exiting/terminating/closing
unexpectedly. If you do see such a line, kill the process (Ctrl+C), and look for
errors.

Once you've verified it works, exit out (Ctrl+C), and run `docker-compose up -d`.
This puts the containers in the background, letting you exit the terminal. Later,
use `docker ps` to check which containers are running (and what they are named),
and `docker logs <container-name>` to check what the output of the command is.

You will need to rotate the accounts periodically, and restart the containers
when that happens.

# Further resources

While the configuration described here is reasonably robust, it is not intended
to be completely tuned and bullet proof. If you bump into problems, these links
will help you find your way around the software stack:

  - [Docker documentation root](https://docs.docker.com/)
  - [Docker Compose Docs](https://docs.docker.com/compose/overview/)
  - [RocketMap Docs](https://rocketmap.readthedocs.io/en/develop/index.html)
  - [PokeAlarm Docs](https://github.com/RocketMap/PokeAlarm/wiki)
  - [MariaDB Learning Site](https://mariadb.org/learn/)
  - [MariaDB Docker Image Info](https://hub.docker.com/_/mariadb/)

# pogo-scanner-docker
Some docker images to make setting up RocketMap slightly easier.

**NOTE:** Support is not guaranteed.

# What does a scalable RocketMap setup look like?
There are several ways to do it. However, not all approaches scale up equally.
First, the major pieces:

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

First, if you haven't [installed docker](https://www.docker.com/community-edition)
already, do that.

Then, copy the `config.ini.example` file somewhere and modify it as needed for
your use. At a minimum, you should change the Google Maps and captcha keys,
database credentials, and webhook settings. See
[the RocketMap docs](https://rocketmap.readthedocs.io/en/develop/first-run/commandline.html)
for more details. All command-line flags should have an equivalent config file
option. Lastly for RocketMap, review `rocketmap/Dockerfile` for a list of
environment variables that need to be set. Figure out which variables need to be
set to what, and save that list somewhere. If you have 5+ areas to scan, that's
going to be a bit of a list. We'll revisit this list in just a bit.

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
in a directory you can mount into a Docker container. No environment variables
needed here, but we'll get back to that directory shortly.

..._Instructions to be continued_...

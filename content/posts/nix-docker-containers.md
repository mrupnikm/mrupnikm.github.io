---
author: "Matic Rupnik"
authorLink: matic-rupnik
date: 2024-04-08
title: Managing Docker containers with Docker Compose in NixOS
tags: [
  "NixOS",
  "Linux",
  "Docker"
]
---
        
I have been warping my head around the easiest implementation of using docker compose in NixOS to best suit my needs for deployments. Turns out there are a lot of ways to do this, but most were not something that would fit my needs that in general include reliable, revertable updates to docker compose files. Thanks to the Jupiter Broadcasting community I have stumbled upon a project conveniently named compose2nix that converts just like the names says docker-compose files into NixOS configurations. So this is a small guide is dedicated to explaining how I use this implementation of it and why I like it. I hope this will help some of you get an idea on if this is a good solution for you as well.


# How it works
Implementing docker compose files into Nix configuration so it is defined on system level is very simple using compose2nix TUI tool. Simply take a docker compose file and run the tool. It will output a docker-compose.nix file that will start a systemd service on startup and deploy the containers. When you want to update the version of your stack you simply change the version of the images in Nix file and commit that to your configuration Nix repository. If there are changes in the underlying compose structure you simply rerun compose2nix to add new changes. Keep in mind that simply updating the container versions would not count as a reliable and safe upgrade of a service so it would be good to think about backing up the volumes first. For this I will explain in more detail at the end as well.

# Example implementation
Using Mealie.io docker compose stack as an example and converting it to Nix configuration:

```yaml
---
version: "3.7"
services:
  mealie:
    image: ghcr.io/mealie-recipes/mealie:v1.3.2 # 
    container_name: mealie
    ports:
      - "9925:9000" # 
    deploy:
      resources:
        limits:
          memory: 1000M # 
    depends_on:
      - postgres
    volumes:
      - mealie-data:/app/data/
    environment:
      # Set Backend ENV Variables Here
      - ALLOW_SIGNUP=true
      - PUID=1000
      - PGID=1000
      - TZ=America/Anchorage
      - MAX_WORKERS=1
      - WEB_CONCURRENCY=1
      - BASE_URL=https://mealie.yourdomain.com

      # Database Settings
      - DB_ENGINE=postgres
      - POSTGRES_USER=mealie
      - POSTGRES_PASSWORD=mealie
      - POSTGRES_SERVER=postgres
      - POSTGRES_PORT=5432
      - POSTGRES_DB=mealie
    restart: always
  postgres:
    container_name: postgres
    image: postgres:15
    restart: always
    volumes:
      - mealie-pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: mealie
      POSTGRES_USER: mealie

volumes:
  mealie-data:
    driver: local
  mealie-pgdata:
    driver: local
```
Note that compose2nix has a few other parameters it accepts that I recommend you take a look before running.

Now for the magic words:
```bash
compose2nix --project mealieio --runtime docker
Generated NixOS config in 2.284698ms
Wrote NixOS config to docker-compose.nix
```
Now you have a file named docker-compose.nix in your directory as it says that looks like so:

```nix
# Auto-generated using compose2nix v0.1.9.
{ pkgs, lib, ... }:

{
  # Runtime
  virtualisation.docker = {
    enable = true;
    autoPrune.enable = true;
  };
  virtualisation.oci-containers.backend = "docker";

  # Containers
  virtualisation.oci-containers.containers."mealie" = {
    image = "ghcr.io/mealie-recipes/mealie:v1.4.0";
    environment = {
      ALLOW_SIGNUP = "true";
      BASE_URL = "https://mealie.yourdomain.com";
      DB_ENGINE = "postgres";
      MAX_WORKERS = "1";
      PGID = "1000";
      POSTGRES_DB = "mealie";
      POSTGRES_PASSWORD = "mealie";
      POSTGRES_PORT = "5432";
      POSTGRES_SERVER = "postgres";
      POSTGRES_USER = "mealie";
      PUID = "1000";
      TZ = "America/Anchorage";
      WEB_CONCURRENCY = "1";
    };
    volumes = [
      "mealie-data:/app/data:rw"
    ];
    ports = [
      "9925:9000/tcp"
    ];
    dependsOn = [
      "postgres"
    ];
    log-driver = "journald";
    extraOptions = [
      "--memory=1048576000b"
      "--network-alias=mealie"
      "--network=mealieio_default"
    ];
  };
  systemd.services."docker-mealie" = {
    serviceConfig = {
      Restart = lib.mkOverride 500 "always";
      RestartMaxDelaySec = lib.mkOverride 500 "1m";
      RestartSec = lib.mkOverride 500 "100ms";
      RestartSteps = lib.mkOverride 500 9;
    };
    after = [
      "docker-network-mealieio_default.service"
      "docker-volume-mealieio_mealie-data.service"
    ];
    requires = [
      "docker-network-mealieio_default.service"
      "docker-volume-mealieio_mealie-data.service"
    ];
    partOf = [
      "docker-compose-mealieio-root.target"
    ];
    unitConfig.UpheldBy = [
      "docker-postgres.service"
    ];
    wantedBy = [
      "docker-compose-mealieio-root.target"
    ];
  };
  virtualisation.oci-containers.containers."postgres" = {
    image = "postgres:15";
    environment = {
      POSTGRES_PASSWORD = "mealie";
      POSTGRES_USER = "mealie";
    };
    volumes = [
      "mealie-pgdata:/var/lib/postgresql/data:rw"
    ];
    log-driver = "journald";
    extraOptions = [
      "--health-cmd='[\"pg_isready\"]'"
      "--health-interval=30s"
      "--health-retries=3"
      "--health-timeout=20s"
      "--network-alias=postgres"
      "--network=mealieio_default"
    ];
  };
  systemd.services."docker-postgres" = {
    serviceConfig = {
      Restart = lib.mkOverride 500 "always";
      RestartMaxDelaySec = lib.mkOverride 500 "1m";
      RestartSec = lib.mkOverride 500 "100ms";
      RestartSteps = lib.mkOverride 500 9;
    };
    after = [
      "docker-network-mealieio_default.service"
      "docker-volume-mealieio_mealie-pgdata.service"
    ];
    requires = [
      "docker-network-mealieio_default.service"
      "docker-volume-mealieio_mealie-pgdata.service"
    ];
    partOf = [
      "docker-compose-mealieio-root.target"
    ];
    wantedBy = [
      "docker-compose-mealieio-root.target"
    ];
  };

  # Networks
  systemd.services."docker-network-mealieio_default" = {
    path = [ pkgs.docker ];
    serviceConfig = {
      Type = "oneshot";
      RemainAfterExit = true;
      ExecStop = "${pkgs.docker}/bin/docker network rm -f mealieio_default";
    };
    script = ''
      docker network inspect mealieio_default || docker network create mealieio_default
    '';
    partOf = [ "docker-compose-mealieio-root.target" ];
    wantedBy = [ "docker-compose-mealieio-root.target" ];
  };

  # Volumes
  systemd.services."docker-volume-mealieio_mealie-data" = {
    path = [ pkgs.docker ];
    serviceConfig = {
      Type = "oneshot";
      RemainAfterExit = true;
    };
    script = ''
      docker volume inspect mealieio_mealie-data || docker volume create mealieio_mealie-data
    '';
    partOf = [ "docker-compose-mealieio-root.target" ];
    wantedBy = [ "docker-compose-mealieio-root.target" ];
  };
  systemd.services."docker-volume-mealieio_mealie-pgdata" = {
    path = [ pkgs.docker ];
    serviceConfig = {
      Type = "oneshot";
      RemainAfterExit = true;
    };
    script = ''
      docker volume inspect mealieio_mealie-pgdata || docker volume create mealieio_mealie-pgdata
    '';
    partOf = [ "docker-compose-mealieio-root.target" ];
    wantedBy = [ "docker-compose-mealieio-root.target" ];
  };

  # Root service
  # When started, this will automatically create all resources and start
  # the containers. When stopped, this will teardown all resources.
  systemd.targets."docker-compose-mealieio-root" = {
    unitConfig = {
      Description = "Root target generated by compose2nix.";
    };
    wantedBy = [ "multi-user.target" ];
  };
}
```
Big scary block of code, right?

Under the hood it will implement oci-containers module from Nix and make a docker-compose-mealieio-root service that can be used to stopping and starting the deployment.

To incorporate this into our Nix configuration we simply add the the nix file into the include block of the main configuration.nix like so:
```nix
{ config, pkgs, ... }:

{
imports =
[
./docker-compose/mealieio/docker-compose.nix
#other imported files...
];
...
```
Now to apply the new Nix OS configuration with Mealie docker implementation.

```bash
sudo nixos-rebuild switch --flake .#jarvis
```
# Automation
What I have demonstrated is all great but it lacks the sort of automated feel that you would expect is possible and that would include a backing up in case of a disaster that can happen on upgrades. Where it is possible to automate the following way in nix as well I decided to go the simpler route as with just a shell script that is run independently from the configuration but is still part of the base git repository.

So lets look at the following simple script using BTRFS subvolume snapshoting as a base of the backup of docker volumes.

```bash
#! /usr/bin/env nix-shell
#! nix-shell -i bash -p bash

VERSION="v1.100.0"
HOST="Server1"

echo "Stopping stack"
sudo systemctl stop docker-compose-mealie-root.target

echo "creating a snapshot"
sudo btrfs subvolume snapshot /data/mealieio /data/.snapshots/mealieio_PRE-${VERSION}

echo "Recreating nix configuration from docker compose"
compose2nix -check_systemd_mounts  -project="mealieio" -runtime="docker"

echo "Rebuilding system"
sudo nixos-rebuild switch  --flake .#${HOST}
```
Just in case you are not familiar with the above commands lets go over them:

first we stop the service running the docker compose that will effectively gracefully shut down all containers
create a snapshot of the docker volume that holds all the data from the containers
recreate the docker-compose.nix from docker-compose.yml that you have edited to fit the upgrade
rebuild your Nix OS configuration(this command is using flakes)
Ending thoughts
This way of going about managing docker compose stacks in Nix OS is by no means the only way you can do it, but I have found this to work best for me and my needs. I hope it can server as an inspiration for your setup.

Special thanks to Assil Ksiksi for creating this library and the Jupiter Broadcasting community.

Happy Nixing!
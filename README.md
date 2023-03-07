This set of scripts aims to create fresh instance of [holo-nixpkgs](https://github.com/Holo-Host/holo-nixpkgs) based Hydra on a Digital Ocean droplet. The process consists of two stages - creating server and restoring configuration from keystore file and backup.

## Ownership Info
Codeowner: @peeech
Consulted: None
Informed: @zo-el

## Creating server
First create new droplet on Digital Ocean with minimum 16GB of RAM and `Ubuntu 16.04 x64` operating system. Use your ssh key for authentication. Choose a datacenter region `Amsterdam 3` and VPC Network `Hydra-ams3`. Also enable ipv6.

Ssh into your new droplet. Exec in shell:
```bash
curl https://raw.githubusercontent.com/Holo-Host/holo-hydra-create/master/holo-hydra-create | bash 2>&1 | tee /tmp/hydra_config.log
```
After successful run system will reboot into newly created empty Hydra.

## Restoring configuration

First import a file `hydra-keystore.enc` to server's `~/` dir. The file is encrypted and password protected as it contains all the security credentials required to configure Hydra. It will be securely erased after successful creation of Hydra. The file itself and encryption password can be obtained from Service Ops, or created from `hydra-keystore-template` and encrypted using command:
```
openssl enc -aes-256-cbc -salt -in hydra-keystore-template -out hydra-keystore.enc
```

Once `hydra-keystore.enc` is in place run
```bash
holo-hydra-restore
```
and watch terminal output for prompts.

You should see the line `Hydra restored from backup successfully`. From now on wait ~1h for hydra to finish evaluations or watch logs with `journalctl -f -u hydra-evaluator` until evaluations are done.

> Important! Recently the way github token is read by hydra has changed. Therefore one additional step is needed:
> Create file `/var/lib/hydra/github_authorizations.conf` with the following content:
```
<github_authorization>
  Holo-Host = token ghp_****
</github_authorization>
```

## Updating TLS certs

There are two urls served from Hydra server: `hydra.holo.host` and `holoportbuild.holo.host`, both via https. Before you switch DNS to newly created machine make sure to copy content of `/var/lib/acme/hydra.holo.host/` and `/var/lib/acme/holoportbuild.holo.host/` from old working Hydra to the one currently created. Once you copy those make sure to restart `nginx.servce` on new Hydra.

## Switching instances
DNS entry `hydra.holo.host` and `holoportbuild.holo.host` are both pointing to Load Balancer hosted on Digitalocean under IP `174.138.104.59`. [Load Balancer Console](https://cloud.digitalocean.com/networking/load_balancers/5024c0aa-2e05-4a2e-acce-2d327aaee036/droplets) shows which instance of Hydra is currently receiving traffic. If you need to switch instances do it in this console, by FIRST removing an old droplet and THEN adding a new one.

> IMPORTANT: Hydra-load-balancer can be pointing only to one instance of Hydra at the same time, otherwise Hydra will enter inconsistent state.

A new instance Status will be showing as `Down` for the first 20s, afterwards it will turn to `Healthy`. During that period `hydra.holo.host` will be down.

## Minions

Hydra needs 2 minions - arm64 and darwin machines. Both of them need to be set up as separate servers.

### Minion 1 - ARM

#### To create minion-1 do what follows:
 - In Holo AWS account (Ohio us-east-2) find AMI "Hydra ARM minion"
 - Based on this AMI launch an instance:
   - t4g.xlarge
   - 200GB General Purpose SSD
   - port 22 open to 0.0.0.0
   - you can choose any key at final dialog

once instance is ready you can ssh to it with any key listed in `<holo-nixpkgs/profiles/logical/holo/default.nix>`. Make sure to change DNS entry `hydra-minion-1.holo.host` and you're good to go.

### Minion 2 - Darwin

Currently minion-2 is hosted on MacStadium.

#### Access existing machine:
 - users listed in `<holo-nixpkgs/profiles/logical/holo/default.nix>` can ssh into minion-2 using `ssh administrator@hydra-minion-2.holo.host -i <your_ssh_key>`
 - everybody else can use VNC to connect to the screen of macOs. Login credentials should be obtained from TechOps

 #### To create minion-2 do what follows:
 - create new machine on MacStadium
 - follow instructions to connect to it over VNC
 - open terminal and via CLI copy keys from `<holo-nixpkgs/profiles/logical/holo/default.nix>` into `/Users/administrator/.ssh/authorized_keys`
 - [install nix](https://nixos.org/download.html)
 - add to `/etc/nix/nix.conf`
 ```
 build-users-group = nixbld
 trusted-users = administrator
 ```
 - reboot
 - update DNS entry for `hydra-minion-2.holo.host`

In case macs from MacStadium were not available you can convert any Mac laptop into minion, as long as it has a static IP and can be accessed over ssh from outside. Just follow above instructions.

## Creating database backup

Hydra stores history of builds in postgres db. If you feel that you need to back up that kind of data `holo-hydra-create-backup` will create backup and upload it to wasabi. Prior to running it fill out `AWS_SECRET_ACCESS_KEY` and `RESTIC_PASSWORD` in a script file.

## Acknowledgements

Based on [nixos-infect](https://github.com/elitak/nixos-infect/blob/master/nixos-infect).

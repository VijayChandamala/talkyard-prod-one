Talkyard production installation
================

For one single server.

You should be familiar with Linux, Bash and Git. Otherwise you might run into
problems. For example, there might be Git edit conflicts, if you and we change
the same file — then you need to know how to resolve those edit conflicts.
Also, knowing a bit about Docker and Docker containers can be good.

This is beta software; there might be bugs.
You can report problems and ask questions in [our support forum](http://www.talkyard.io/forum/latest/support).

If you'd like to install on your laptop / desktop just to test, there's
[a Vagrantfile here](scripts/Vagrantfile) — open it in a text editor, and read,
for details.

Installation overview: You'll rent a virtual private server (VPS) somewhere, then download
and install Talkyard, then sign up for a send-emails service and configure settings,
then optionally configure OpenAuth login for Google, Facebook, Twitter, and GitHub. And
preferably configure off-site backups. — These steps aren't unique to Talkyard. Instead
it's the same for all self hosted discussion software with social login and emails.

We'll soon write instructions about how to configure email settings and Google &
Facebook & GitHub & Twitter login. **Send us an email** (address below) and we'll
tell you when we're done writing the instructions: (you could also mention
if you want to be one of the first few people who try to follow those instructions, or not)

`hello at talkyard.io`


Get a server
----------------

Provision an Ubuntu 18.04 server with at least 2 GB RAM. Here are three
places to hire servers:

- Digital Ocean: https://www.digitalocean.com/ — easy to use.

- Google Compute Engine: https://cloud.google.com/compute/
  — for advanced users. You should be a company, because Google says you should pay taxes yourself.

- Scaleway, https://www.scaleway.com/ — inexpensive, just €3, but risky, not so good backups.
  The server should be X86-64, not ARM (so don't use the BareMetal ARM servers).


Installation instructions
----------------

1. Become root and install Git:

        sudo -i
        apt-get update
        apt-get -y install git

1. Download installation scripts: (you need to install in
   `/opt/talkyard/` for the backup scripts to work)

        cd /opt/
        git clone https://github.com/debiki/talkyard-prod-one.git talkyard
        cd talkyard

1. Prepare Ubuntu: install tools, enable automatic security updates, simplify troubleshooting,
   and make ElasticSearch work:

        ./scripts/prepare-ubuntu.sh 2>&1 | tee -a talkyard-maint.log

   (If you don't want to run all stuff in this script, you at least need to copy the
   sysctl `net.core.somaxconn` and `vm.max_map_count` settings in the script to your
   `/etc/sysctl.conf` config file — otherwise, the full-text-search-engine (ElasticSearch)
   won't work. Afterwards, run `sysctl --system` to reload the system configuration.)

   Also do this, to avoid harmless but annoying language-missing warnings:

        export LC_ALL=en_US.UTF-8

1. Install Docker:

        ./scripts/install-docker-compose.sh 2>&1 | tee -a talkyard-maint.log

1. Start a firewall: (and answer Yes to the question you'll get. You can skip this if
   you use Google Cloud Engine; GCE already has a firewall)

        ./scripts/start-firewall.sh 2>&1 | tee -a talkyard-maint.log

1. Edit config files:

        nano conf/app/play.conf   # edit all config values in the Required Settings section
        nano .env                 # edit the database password

   Note:
   - If you don't edit `play.http.secret.key` in file `play.conf`, the server won't start.
   - If you're using a non-standard port, say 8080 (which you do if you're using **Vagrant**),
     then add `talkyard.port=8080` to `play.conf`.

1. Depending on how much RAM your server has (run `free -mh` to find out), choose one of these files:
   mem/1g.yml, mem/2g.yml, mem/3.6g.yml, ... and so on,
   and copy it to ./docker-compose.override.yml. For example, for
   a server with 2 GB RAM:

        cp mem/2g.yml docker-compose.override.yml

1. Install and start the latest version. This might take a few minutes
   the first time (to download Docker images).

        # This script also installs, although named "upgrade–...".
        ./scripts/upgrade-if-needed.sh 2>&1 | tee -a talkyard-maint.log

1. Schedule deletion of old log files, daily backups and deletion old backups,
   and automatic upgrades:

        ./scripts/schedule-logrotate.sh 2>&1 | tee -a talkyard-maint.log
        ./scripts/schedule-daily-backups.sh 2>&1 | tee -a talkyard-maint.log
        ./scripts/schedule-automatic-upgrades.sh 2>&1 | tee -a talkyard-maint.log

1. Point a browser to the server address, e.g. <http://your-ip-addresss> or <http://www.example.com>
   or <http://localhost>. Or <http://localhost:8080> if you're testing with Vagrant.
   In the browser, click _Continue_ and create an admin account
   with the email address you specified when you edited `play.conf` earlier (see above).
   (Google and Facebook login won't work yet, because you have not
   configured OpenAuth settings in `play.conf`.)

1. No email-address-verification-email will be sent to you, because you have not
   yet configured any email server (in `play.conf`).  However, you'll find
   an address verification URL in the application server's log file, which you can view
   like so: `./view-logs app` (or `./view-logs -f --tail 30 app`). Copy-paste
   the URL into the browser.  You can [send an email again] / [write the URL to the log file
   again] by clicking the _Send email again_ button.


Now you're done. Everything will restart automatically on server reboot.

Next things for you to do:

- In the browser, follow the getting-started guide.
- Send an email to `hello at talkyard.io` so we get your address, and can
  inform you about security issues and major software
  upgrades that might require you to do something manually.
- Copy backups off-site, regularly. See the Backups section below.
- Pay for some send-email-service (websearch for "transactional email services")
  and configure email server settings in `/opt/talkyard/conf/app/play.conf`.
- Configure Gmail and Facebook login:
  - At Google, Facebook, GitHub and Twitter, login and create OpenAuth apps,
    then add the API keys and secrets to `play.conf` — I should write
    instructions for this.



Upgrading to newer versions
----------------

If you followed the instructions above — that is, if you ran these scripts:
`./scripts/configure-ubuntu.sh` and `./scripts/schedule-automatic-upgrades.sh`
— then your server should keep itself up-to-date, and ought to require no maintenance.

In a few cases you might have to do something manually, when upgrading.
Like, running `git pull` and editing config files, maybe running a shell script.
For us to be able to tell you about this, please send us an email at
`hello at talkyard.io`.

If you didn't run `./scripts/schedule-automatic-upgrades.sh`, you can upgrade
manually like so:

    sudo -i
    cd /opt/talkyard/
    ./scripts/upgrade-if-needed.sh 2>&1 | tee -a talkyard-maint.log



Backups
----------------

### Importing a backup

You can import a Postgres database backup like so: (you need to stop the 'app'
container, otherwise the import will fail because of active database
connections)

    sudo -i
    docker-compose stop app
    zcat /opt/talkyard-backups/BACKUP_FILE.gz \
      | docker exec -i $(docker-compose ps -q rdb) psql postgres postgres \
      | tee -a talkyard-maint.log
    docker-compose start app

Replace `BACKUP_FILE` above with the actual file name.

TODO: Explain how to import the uploaded-files backup archive...

You can login to Postgres like so:

    sudo docker-compose exec rdb psql postgres postgres  # as user 'postgres'
    sudo docker-compose exec rdb psql talkyard talkyard  # as user 'talkyard'


### Backing up, manually

You should have configured automatic backups already, see the Installation
Instructions section above. In any case, you can backup manually like so:

    sudo -i
    cd /opt/talkyard/
    ./scripts/backup.sh manual 2>&1 | tee -a talkyard-maint.log


### Copy backups elsewhere

You should copy the backups to a safety backup server, regularly. Otherwise, if your main server suddenly disappears, or someone breaks into it and ransomware-encrypts everything — then you'd lose all your data.

See [docs/copy-backups-elsewhere.md](./docs/copy-backups-elsewhere.md).



Tips
----------------

If you'll start running out of disk, one reason can be old patches for automatic operating system security updates.
You can delete them to free up disk:

```
sudo apt autoremove --purge
```


Docker mounted directories
----------------

- `conf/`: Container config files, mounted read-only in the containers. Can add to a Git repo.
- `data/`: Directories mounted read-write in the containers (and sometimes read-only too). Not for Git.



License (GPLv2)
----------------

The GNU General Public License, version 2 — and it's for the instructions and
scripts etcetera in this repository only, not for any Talkyard
source code or stuff in other repositories.

    Copyright (c) 2016-2018 Kaj Magnus Lindberg

    This program is free software; you can redistribute it and/or modify
    it under the terms of the GNU General Public License as published by
    the Free Software Foundation, version 2 of the License.

    This program is distributed in the hope that it will be useful,
    but WITHOUT ANY WARRANTY; without even the implied warranty of
    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    GNU General Public License for more details.

    You should have received a copy of the GNU General Public License along
    with this program; if not, write to the Free Software Foundation, Inc.,
    51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA.

Here's the full license text: [LICENSE-GPLv2.txt](./LICENSE-GPLv2.txt).


<!-- vim: set et ts=2 sw=2 tw=0 fo=r list : -->

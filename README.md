# Ansible running jitsi's `docker-compose up` with podman-compose

Following jitsi's [Self-Hosting Guide - Docker](https://jitsi.github.io/handbook/docs/devops-guide/devops-guide-docker) using podman I fell down a rabbit hole - this is how I got out. I just followed the *Quick start* up to point 6 (no `jigasi`, no `etherpad`, no `jibri`).

Problem: 
* `podman-compose` does not honor the `networks:...` part in a docker-compose file (podman-compose [issue #288](https://github.com/containers/podman-compose/issues/288)). This results in unknown hence unreachable upstreams in the http proxy / nginx.
* Some environment variables missing [comment in issue #201](https://github.com/jitsi/docker-jitsi-meet/issues/201#issuecomment-609023954). I do not know if this is a `podman-compose` exclusive issue but the result is "unfinished" config templating.

The good news is that if you followed the self-hosting guide, just run `podman-compose up` with some extra parameters and you should be good to go:
```
[bob@somehost docker-jitsi-meet-stable-6433]$ podman-compose --podman-run-args "--env-file .env --add-host xmpp.meet.jitsi:127.0.0.1" up -d
['podman', '--version', '']
using podman version: 3.4.1-dev
** excluding:  set()
podman pod create --name=docker-jitsi-meet-stable-6433 --share net --infra-name=docker-jitsi-meet-stable-6433_infra -p 4443:4443 -p 80:80 -p 10000:10000/udp -p 443:443
d2d1dcdc2f92e43ea4ac50b0b9bf9d99ab11e34ddf3a712b32fcd324cdafa5be
exit code: 0
[...]
``` 

The other good news is that I wrapped all steps in ansible ("lazy as fuck as a service"). 
This will start a basic jitsi instance from the docker-compose tarball using `podman-compose`. 
The instance will run as an unprivileged user on ports 80 and 443 including redirect and LetsEncrypt certificate.

### Disclaimer - this is intended for debugging or demonstration purposes only
This is in no way supposed to be a productive setup. Use at own risk - amend the vars in `playbook.yml` and give it a try (using your own inventory).

If you are testing the setup I recommend to use the LetsEncrypt staging API. LE API rate limiting opens another debugging rabbithole you may not want fall into - I have been there for you already.

```
user@box:~/ansible_podman-compose_jitsi$ ansible-playbook -i <inventory> playbook.yml 

PLAY [all] ********************************************************************************************

TASK [Gathering Facts] ********************************************************************************
ok: [somehost]

TASK [Install podman and python] **********************************************************************
ok: [somehost]

TASK [Allow podman privileged ports for non root users] ***********************************************
ok: [somehost]

TASK [Add jitsi_user] *********************************************************************************
ok: [somehost]

TASK [install 'podman-compose' for jitsi_user] ********************************************************
ok: [somehost]

TASK [Generate directories] ***************************************************************************
ok: [somehost] => (item=web/crontabs)
ok: [somehost] => (item=web/letsencrypt)
ok: [somehost] => (item=transcripts)
ok: [somehost] => (item=prosody/config)
ok: [somehost] => (item=prosody/prosody-plugins-custom)
ok: [somehost] => (item=jicofo)
ok: [somehost] => (item=jvb)
ok: [somehost] => (item=jigasi)
ok: [somehost] => (item=jibri)

TASK [Download jitsi release] *************************************************************************
ok: [somehost]
[....]

```

The tasks will 
* install podman
* **allow unprivileged users to use port numbers starting from 80** (sysctl)
* add a unprivileged user
* install podman-compose for that user
* follow instructions from [Self-Hosting Guide - Docker](https://jitsi.github.io/handbook/docs/devops-guide/devops-guide-docker)
* start the instance with `podman-compose up ...`

A note on idempotency: The playbook is tested and I am at least 85% confident that I covered most cases but... you'll never know and have been warned ;)

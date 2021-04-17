---
title: "AppArmor Gotcha"
date: 2021-04-16T08:59:40Z
draft: false
---

> I use Ubuntu. Mileage may vary on other distributions

## Protecting Secrets in Home With AppArmor

Plenty of applications stash secrets in sub-directories of home. You may
not even notice as the directories are sometimes hidden e.g ~/.aws may contain
API keys for AWS. Firefox, for example, has
no real need to access API keys for cloud services
so I have added the following to /etc/apparmor.d/local/usr.bin.firefox:

```shell
# Site-specific additions and overrides for usr.bin.firefox.
# For more details, please see /etc/apparmor.d/local/README.

# covers google and firebase credentials
deny @{HOME}/.config/ rwmkl,
deny @{HOME}/.config/** rwmkl,
# aws has its own directory
deny @{HOME}/.aws/ rwmkl,
deny @{HOME}/.aws/** rwmkl,
```

When I use Firefox to browse files in my home dir, permission to access ~/.aws is denied because AppArmor prevents access as intended:

![firefox permission denied](/images/apparmorbypass/denied.png#center)

The idea is to limit the impact of Firefox
being compromised.

## Paths and Namespaces

I use [rootless docker](https://docs.docker.com/engine/security/rootless/) which requires [user
namespace](https://man7.org/linux/man-pages/man7/user_namespaces.7.html) functionality to operate without
root. It's a huge improvement over the usual
insecure workflow with docker where any program running as an unprivileged user
can start up a container running as root with access to all system resources. As an extra layer of security, I was interested in writing an apparmor profile for docker
and the containers it launches which would
protect secrets in my home directory.

To test the profile, I made of copy of bash called "bosh" and wrote a basic profile which would
resemble what I had in mind for docker:

```shell
#include <tunables/global>
profile bosh /home/vagrant/bosh {
  # Allow all rules
  capability,
  mount,
  file,
  unix,
  ptrace,
  /dev/pts/* rw,
  # aws has its own directory
  audit deny @{HOME}/.aws/{,**} mrkwl,
}
```

Docker needs to be able to do bind mounts and this causes us
problems: the directory ~/.aws can be bind mounted to, say, ~/target/. The contents of ~/.aws will be available at ~/target and, crucially, will be considered by AppArmor to not match
the deny rule

```shell
audit deny @{HOME}/.aws/{,**} mrkwl,
```

Here's an example of (ab)using user namespaces to access secrets
in ~/.aws:

![user namespace bypass](/images/apparmorbypass/mount.gif#center)

Here, a user namespace is created
from our "bosh" shell. ~/.aws is bind mounted
to ~/target and the file "foo" is copied. So,
if docker (or anything which can run docker) is compromised, then the AppArmor profile
isn't enough to protect files in ~/.aws.

## Docker Impacts AppArmor Rules

If you have non-rootless docker installed and your user has access to the docker socket,
the user has root access. For example, by running

```shell
docker run -it -v /:/root alpine /bin/sh
```

you drop into a shell as root with the host computer's root-filesystem available at /root.
You can now do whatever you like to the system: steal files, reset passwords,
create backdoor accounts, modify local logs to hide the intrusion etc.

With rootless-docker, the user in the container shell isn't root but the normal user account, even if "whoami" claims otherwise:

```
/ # whoami
root
/ # cat /root/etc/shadow
cat: can't open '/root/etc/shadow': Permission denied
/ # 
```

The user is called "root" inside the container but is mapped to our normal, unprivileged user account and so /root/etc/shadow is off limits.

Although rootless-docker is an improvement from a security point of view, AppArmor profile
writers should be aware that access to the docker socket (/run/user/1000/docker.sock on my
system) allows file access rules to be bypassed. In the running example of protecting ~/.aws,
running the following (or driving the docker API in an equivalent way)

```shell
docker run --rm -v "$HOME":/homecopy alpine /bin/sh -c "cp -R /homecopy/.aws /homecopy/awscopy"
```

copies the directory ~/.aws to ~/awscopy and the secrets therein are no longer protected by
the AppArmor profile.

> AppArmor profile writers should restrict access the docker socket unless it is required. If it is required, it should be noted that pathname based restrictions can be bypassed.

## Is This A Bug?

I think it's at least a very sharp-edge you could cut yourself on when writing AppArmor profiles
when user namespaces and bind mounts are in the mix.

Maybe the whole concept of controlling access by the filename is flawed? Maybe SELinux labels are
more appropriate for modern container setups?















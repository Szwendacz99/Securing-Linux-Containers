# Securing Linux Containers

# 1. Table of contents

<!--toc:start-->
- [Securing Linux Containers](#securing-linux-containers)
- [1. Table of contents](#1-table-of-contents)
- [2. Introduction](#2-introduction)
- [3. Secrets](#3-secrets)
  - [3.1 Alternatives](#31-alternatives)
    - [3.1.1 Files](#311-files)
    - [3.1.2 Secrets Management Services (kubernetes)](#312-secrets-management-services-kubernetes)
- [4. Users and groups](#4-users-and-groups)
  - [Setting user and group](#setting-user-and-group)
    - [Containerfile/Dockerfile](#containerfiledockerfile)
    - [Changing user/group arbitrarily on container startup](#changing-usergroup-arbitrarily-on-container-startup)
  - [Additional security](#additional-security)
- [5. Filesystem](#5-filesystem)
- [6. Resources limits](#6-resources-limits)
- [7. Network](#7-network)
- [8. Images](#8-images)
  - [8.1 Building](#81-building)
  - [8.2 Scanning](#82-scanning)
- [9. Selinux](#9-selinux)
<!--toc:end-->

# 2. Introduction

This document is a collection of simple and very generic tips and best
practices related to seciurity of Linux containers. Contenerization is
considered safer by default, but then one can hear about discovered
vulnerabilities that are primarly bad for applications in containers
(Example: [CVE-2023-49103](https://nvd.nist.gov/vuln/detail/CVE-2023-49103)).
Tips and best practices collected here should help raise awarness about
how to keep containers really secure. Contents are kept container-engine
agnostic, but examples will be based on actual implementations (Podman, k8s).

# 3. Secrets

Secret is the most vulnerable data, as it usually can open access to other
private data. They might also allow modification of the environment, which
means possibilities for further access or many other forms of attack.

> [!WARNING]
> Don't use environment variables for secrets

Container isolation made providing and managing secrets somewhat harder, as
they need to cross the additional barier. This casued the rather dangerous
trend of providing secrets among many other configuration data in form of
environment variables. At first sight it might look like good idea, but when
actually compared to other means of storing secrets it turns out that
environment variables might be much easier to access by attacker, than
for example arbitrary files. [CVE-2023-49103](https://nvd.nist.gov/vuln/detail/CVE-2023-49103)
is only an example of vulnerability which was considered to be more
dangerous for contenerized apps, because of the vulnerability
being based on gaining access to env variables.

## 3.1 Alternatives

### 3.1.1 Files

Files with secrets are common and broadly supported. With proper setup they can
be also very secure.

- Keep configuration and secret files on entirely different path than other data
- If application runs main process under different user than worker processes
(which usually have direct contact with user interaction), the configuration
should not be readable by the worker process user.
- Depending on the technology used, storage of the secret files inside of a
container could be temporary/volatile. In kubernetes Secret objects are mounted
as tmpfs. Example for mounting secret as tmpfs in pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: registry.fedoraproject.org/fedora-minimal:latest
    command: [ "sleep", "infinity" ]
    volumeMounts:
      - mountPath: /config
        name: config
  volumes:
  - name: config
    secret:
      secretName: config
```

This produces readonly tmpfs mount inside:

```bash
bash-5.2# df -h /config/
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           4.8G  4.0K  4.8G   1% /config

bash-5.2# ls -la /config/
total 0
drwxrwxrwt. 3 root root 100 Nov  9 14:00 .
drwxr-xr-x. 1 root root  24 Nov  9 14:00 ..
drwxr-xr-x. 2 root root  60 Nov  9 14:00 ..2024_11_09_14_00_47.4065932771
lrwxrwxrwx. 1 root root  32 Nov  9 14:00 ..data -> ..2024_11_09_14_00_47.4065932771
lrwxrwxrwx. 1 root root  18 Nov  9 14:00 secret.conf -> ..data/secret.conf
```

### 3.1.2 Secrets Management Services (kubernetes)

There are sophisticated tools for secret management and their deployment,
available for kubernetes. For example HashiCorp Vault. It offers dynamic
secrets, secret rotation, and access policies. Such tools are most helpfull in
large environments and infrastructures, where secret management is split
among many people.



# 4. Users and groups

Users and groups are standard mechanisms for security and permissions limiting
in unix-like systems. Contenerization engines usually have possibility to
arbitrarily assign them to the contenerized program process.

> [!NOTE]
> Both user and group can always be specified by numeric id even if no actual
> user or group is assigned to them. When specifying with string name, the user
> or group must exist **inside** of the container (`/etc/passwd`, `/etc/group`)

> [!NOTE]
> Processes of rootless containers or containers with uid/gid mapping have
> different id's inside of container and outside. This can complicate things
> even more, but that also usually greatly increases security.
> In some scenarios such mapping can also cause trouble with files in
> container image, if their id's are out of mapping range.

## Setting user and group

Containers have default user and group specified by Containerfile, but
it can be changed when starting the container.

### Containerfile/Dockerfile

In Containerfile the user/group assignment might take place many times in
single build. Typical reason for that is to have high privilige (root) during
build, and then set default to unpriviliged user at the end of build, so that
containers will use it by default.

Setting just user to "user1"

```Dockerfile
USER user1
```

Setting both user and group
```Dockerfile
USER user1:group1
```

Setting just group
```Dockerfile
USER :group1
```

### Changing user/group arbitrarily on container startup

Podman and Docker uses `--user` or shorter `-u` flag to specify both user and
group. The syntax is the same as shown for Containerfile. Example:

```bash
❯ podman run --rm -it --user 1:bin registry.fedoraproject.org/fedora-minimal
bash-5.2$ whoami
bin
bash-5.2$ groups
bin
bash-5.2$ grep ^bin /etc/passwd
bin:x:1:1:bin:/bin:/usr/sbin/nologin
bash-5.2$ grep ^bin /etc/group
bin:x:1:
```

For Kubernetes, the user and group specification is located in pod definition:

```yaml
apiVersion: v1
kind: Pod
spec:
  securityContext:
    runAsUser: 1
    runAsGroup: 1
```

> [!NOTE]
> In kubernetes you can't specify user nor group using string name.
> Only numeric values are allowed.

## Additional security

Linux kernel provides usefull feature - [No New Privileges Flag](https://docs.kernel.org/userspace-api/no_new_privs.html).
If set for process, it prevents the process from gaining more privileges than
parent process. This effectively blocks use of capabilities, and setgid,setuid
flags on files, which are known and powerfull tools for exploitation.

In Podman and Docker, the flag can be enabled using parameter `--security-opt no-new-privileges`

In Kubernetes, there is section related to security context per container:

```yaml
(....)
  containers:
  - name: mycontainer
    securityContext:
      allowPrivilegeEscalation: false
(....)
````

# 5. Filesystem

# 6. Resources limits

# 7. Network

# 8. Images

## 8.1 Building

## 8.2 Scanning

# 9. Selinux

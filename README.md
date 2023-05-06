## Docker security - Users

### Instructions

When running processes inside the container, we should follow best practices when it comes to security, just like when running them on the host machine. That means creating a non-root user, assigned only the necessary capabilities, which we will use to run our programs. Be careful, when creating a user, you **must** assign a new `UID` to it. Always choose an id above `10,000`, unless you're trying to do something specific. 

New users are created with different commands, depending on which base os image we are using. For the purposes of this guide, we'll be using `alpine` linux, and one example of user creation would be
```
RUN addgroup --gid 10001 \
             example-group \
 && adduser --disabled-password \
            --gecos "Example user" \
            --shell /bin/sh \
            --home /home/example-user \
            --ingroup example-group \
            --uid 10000 \
            example-user
```

`gid`/`uid` - Set the id of the new group/user \
`disable-password` - Don't assign a password \
`gecos` - User's full name \
`shell` - Login shell i.e. shell that starts when you log into this user \
`home` - Home directory path \
`ingroup` - Users primary group. If `ingroup` isn't specified, `GID` will match `UID`. In case `GID` with the same value as the provided `UID` already exists, the command will fail.

After creating a new user in `Dockerfile`, you can tell Docker to run all subsequent commands under this user, by using the `USER` command. However, one important thing to note, is that `COPY` and `ADD` commands ignore `USER`. This means, all files will be added by `root` user. In order to prevent that, use `--chown` flag to change their ownership. This is preferred over changing the ownership after copying the files, because of Docker layering system, which will double the size of copied files, simply for changing their ownership.

### Explanation

#### Containarization

Above mentioned instructions raise a few questions. Why do we need to create a non-root user even though we are working inside a container? Isn't the user isolated within the container, having no access to the underlying host system? Furthermore, why do we need to specify `UID`, and why should it be greater than 10000? In order to answer those questions, we should understand how Docker works.

There is no need to explain the whole architecture in detail, but one thing to take away is that Docker doesn't actually do any containarization itself, rather, it uses containarization feature built into the `Linux kernel`. That's the reason `Docker engine` works on linux system only. If you run `Docker engine` on `Windows`, or `MacOS`, docker will actually spin up a `VM` containing alpine linux, in order to use its containarization.

#### Linux namespaces

Containarization is supported in `Linux` by `namespaces`. `Namespaces` provide isolation of resources on the kernel level. This is a way of separating processes from each other, where, in a way, a process inside a namespace is tricked into thinking it has its own instance of the global resource. \
There are 8 different types of namespaces so far:

`Mount (mnt)` - controls mount points \
`Process ID (pid)` - controls ID's of processes \
`Network (net)` - virtualizes the network stack \
`Inter-process Communication (ipc)` - isolate processes from System V style IPC \
`UTS` - allows for multiple different host and domain names \
`User ID (user)` - isolates user ID's and group ID's \
`Control group (cgroup)` - limits and isolates the resource usage of a collection of processes \
`Time` - allows for multiple different system times

#### User namespace

Docker uses most of the above mentioned namespaces in order to achieve proper containarization of processes. However, to better understand users within the container, we only need to focus on `User namespace`'s.

`User namespace`'s provide us with a way to map users in the namespace to different users in the host. In particular, this allows us to map a root inside a namespace to a non-root user on the host system.

As an example, we can create our own namespace. I'll run it with my everyday user - `reljinm`.
```
unshare --user --map-root-user /bin/bash
```
`--user` - create a user namespace \
`--map-root-user` - map `UID` of user which created namespace to root user inside namespace

Once inside namespace, we can check that we are indeed root, by running `whoami`, or `id` commands. 

![Create namespace](https://github.com/reljadev/docker-users/blob/master/create-namespace.png?raw=true)

However, that doesn't mean that we have root privileges on the host system. That's because our root user is mapped to the user which created the namespace - `reljinm` user. This means that we only have `reljinm` user privileges, which can be confirmed by trying to remove `/bin/bash` for example.

![Remove bash](https://github.com/reljadev/docker-users/blob/master/remove-bash.png?raw=true)

In addition, any file or process created within the namespace will have `root` ownership inside the namespace, however, outside the container the owner will again be `reljinm` user.

![File owner](https://github.com/reljadev/docker-users/blob/master/file-owner-2.png?raw=true)

#### Users within Docker container

What we've learned so far, seems to only reinforce our view that a non-root user within the container isn't necessary, since root user will be mapped to a less privileged user on the Docker host. However, this isn't true, because Docker doesn't use `User namespace` by default. Meaning, **users within the container are the same users on the docker host**. Specifically, root user inside container equals root user outside of it. Nontheless, it's more secure to run programs inside the container, because they are still isolated by all the other namespaces.

Why Docker chose not to use remapping of `UID`/`GID` is not clear to me, however one possible explanation is that enabling user namespaces can cause issues with file permissions when mounting volumes from the underlying host, as the `UID`/`GID` in use in the container may not have rights to mounted directories. If you'd like to enable remapping yourself, you can, by following official docker [instructions](https://docs.docker.com/engine/security/userns-remap/).

Now that it's clear we need to create a non-root user, we can just add one with `adduser` command and, since new users have no elevated access permissions, we are done, right? Not quite.

When new user is created, by default, it's `UID` will be the first available id, greater than 999, because `0-999` range is reserved for system accounts. The caveat here lies in the way `adduser` checks which `UID` is available.

You might think that `adduser` asks the kernel which `UID`'s are not already in use, and then chooses the smallest one from them. But that's not the case. Instead it refers to a `/etc/passwd` file, which is just a plain-text list of all registered users, complete with their usernames, passwords, `UID`'s and more. This file is not controlled by the kernel, but by a third-party software, and it exists simply for our convenience, so we can use named users instead of a bunch of numbers. It's important to realize that `/etc/passwd`, doesn't have to contain all the users present on the system.

In a way there isn't a place where all users on the system are stored. That's because to kernel, user and group ID's are just numbers attached to a process, which are used to see if the process is allowed to read, write or execute a file, that belongs to specific owner (`UID`) and group (`GID`). That's all kernel knows and cares about - `UID`'s and `GID`'s. It doesn't know any usernames.

How do `kernal` and `passwd` interact with each other?  When you log in and give your username, a program, running as root, takes the username and looks up the UID in `/etc/passwd`, asks for the password and checks it. If all goes well, the program changes to that `UID`/`GID` pair and executes the user's login shell.

Normally, all of this works perfectally well, and you can only map one `UID` to only one username. However, because `/etc/passwd` isn't maintained by the kernel, but by a third-party program, operating system within the docker container will come with it's own `/etc/passwd`, and a second mapping to the same `UID` is possible, and it's exactly what happens in practice if you are not careful.

Let's look at an example. Say we create a new Docker image from the following `Dockerfile`
```
FROM alpine:3.17

RUN adduser --disabled-password example
USER example

CMD ["sleep", "infinity"]
```
As you can see, `example` user ran the `sleep` command, and we can check that indeed that process is owned by `example` user inside the container.

![Proces in container](https://github.com/reljadev/docker-users/blob/master/process-in-container-2.png?raw=true)

On docker host then, you would expect to see just the `UID` of `example` user as the owner of that process, since `example` doesn't exist on host. However, you'd be wrong.

![Proces on host](https://github.com/reljadev/docker-users/blob/master/process-on-host.png?raw=true)

What's happening here is that `adduser` within the container checks `/etc/passwd` within the container, and since this is a brand new os, and no users are registered yet, first avaiable `UID` is going to be `1000`. Now we have a user `example` with id `1000` within the container, but since docker doesn't use `User namespaces` that's the same user with id `1000` as on the host, which in my case already exists and has name `pomocnik`. This is a problem, because `pomocnik` might have elevated permission rights, and `example` will have those same rights. In order to avoid that, and trully create a new, previously non-existent user, we should always set `UID` when creating a user, and the value should be high, above `10,000`. This way, we know that id won't overlap with id's of existing users.

#### Implicit `root` rights

One last thing to keep in mind is that adding user to `docker` group, so you wouldn't need to  preface the `docker` command with `sudo`, is potentially very dangerous. In fact adding user to `docker` group is equivalent to giving it permanent, non-password-protected root access. This is a direct consequence of what we've talked about, i.e. root inside the container is the same as root on the host, and the fact that you can mount any directory in the container. For example, following command
```
docker run -v /:/host -it alpine:3.17 /bin/sh
```
would start the container with the `root` user and your whole filesystem mounted inside of it, giving you unrestricted access to `/` directory. That's why you have to be **very careful**, when adding users to `docker` group.

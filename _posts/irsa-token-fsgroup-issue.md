---
layout: post
title: IRSA Token Permission Issue with Kuberentes fsGroup
author: Eunchan Lee
categories: kubernetes
---

# pganalyze-collector

fsGroup was supposed to be 1000 as its container runs with changing its own uid/gid.

As I mentioned above, IRSA is a file. And all files in Linux have permissions. UID and GID.

Originally the irsa token is assigned for only root(`600`). And this cannot be altered.

BTW we can tune it via `securityContext.fsGroup` . and it should be set to exactly the container running user. WAIT. **container running user?**

Yeah, container has its own namespace and every container will be launched as `root` obviously. And you know, the token will be served with permission `600` so hmmmm no problem I guess?

But it ain't be assured in every containers. `pganalyze-collector` is excatly that reference.

Before I'm going to tell you what happened in that container, we, of course, can set uid and gid forcely. but.. what if the container `entrypoint` runs some commands sorta changing its uid/gid? 

Does it sound unnatural or rare? NO WAY. It happened on my fifth deployment.

So we should figure out in which uid/gid the container's `entrypoint` runs.

What about `fsGroup: 65534` ? yeah you might find it on github. Especially `aws external-dns` issue. but.. it's not a master key. that container runs as `65534` which means `nobody`. 

So.. to sum up we should set up `fsGroup` correctly.

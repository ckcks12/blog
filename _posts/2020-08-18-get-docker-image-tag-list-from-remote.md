---
layout: post
title: Get Docker Image Tag List From Remote
author: Eunchan Lee
tags: [programming,docker,ci/cd]
---

# TL;DR
```bash
curl -X GET https://hub.docker.com/v2/repositories/nginx/nginx-ingress/tags/ 2>/dev/null | jq -rM '.results[].name'
```

# Explanation
Using [Docker API v2](#Docker-v2-api), we can query the metadata of images which contains its tag list.

# When to use
Tagging for semantic versioning without deploy pipelines, must be annoying.
So I wanted to print out image tags in my cli or somewhere so that I could track my lastest version easily.

![img](https://user-images.githubusercontent.com/12825679/90426703-78208900-e0fc-11ea-938f-cb5edbe1e073.png)

# Docker v2 api
You can find it in official page: https://docs.docker.com/registry/spec/api/

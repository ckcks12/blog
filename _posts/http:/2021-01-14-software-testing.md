---
layout: post
title: Software Testing
author: Eunchan Lee
categories: [DevOps,Testing]
---

Software is splitted into two parts: CRUD and Computation.

Let's have an example. Notion. When you type, it reads your input and create or update it on Database and then create UI from it to you. So we can think its main job is somehow related to CRUD. However you might think that it uses External APIs and it's not a kind of CRUD-able. I also thought like that at first glance but External APIs are also CRUD internally. So an app using external CRUD apps can be CRUD app. What about Business Logic? It can be Computation in CRUD app. It's quite obvious since our codes go down to CPU and what cpu does is computation with ALU unit.

Software Testing is a process which assures us of software's quality, performance and stability. And when it comes to the two parts, CRUD and Computation, it's getting clear.

![image](https://user-images.githubusercontent.com/12825679/104560053-aba21800-5688-11eb-8afd-a2251b2c7f5d.png)


For CRUD, let's assume we're going to test **Read from DB** feature. Code would be consist of Connecting, Querying, Fetching and Print to user. In this case we have three approach for testing.

1. Mock Service

    Mocking service might seems quite similar to Unit Testing(but it's not Unit Testing). Mock Service does what each functions(features) dependencies do in a way we told it to do. In this example Mock Service might be Repository Service and we coded it to return this array and this values **always**. This leads us be unable to test our repository service code itself since it is being mock-ed. What if we had problems such as Wrong SQL Syntax, SQL Function Usage, Database Type Casting (scanning nullable array of nullable string took my 3 nights). 

    Pros:

    - Most convinient and fastest
    - Less code

    Cons:

    - Not coverage 100%

2. Mock DB (Alpha DB)

    Mock DB usually be implemented by Docker Compose. Blessed us who can startup PostgresDB with a single line of code. And it's fully functioning service which helps us testing coverage 100%. However, some services are difficult to setup and some are impossible(In-house Micro Services, vendor-specific services(BigQuery) and etc.)

    Pros:

    - Fully functioning
    - Coverage 100%

    Cons:

    - Difficulity in setup
    - Impossible of mock Micro Services/Not open source projects
    - Heavy Setup like Data Intensive(Batch insertion, etc.) → test be slower

3. Mock Nothing

    Mock Nothing means that just use the services out there. They could be dev, alpha and even production environment. In the case of production environemtn, it acts just like monitoring test. It's also fully funcitoning services and easier than Mock DB. However it runs on full set of environment and there might be problems somewhere not our code. 

    Pros:

    - Micro Services, BigQuery etc. can be provided
    - Heavy Setup like Data Intensive also be prepared always (or just faster)
    - Faster than Mock DB (don't need to prepare everytime)

    Cons:

    - Environment is vulnerable
    - Environment is out of control
    - When test failed, might not code problem

To sum up, here's a table.

||Mock Service|Mock DB|Mock Nothing|
|-|:-:|:-:|:-:|
|Amount of code for implementation|A lot|A little|Zero|
|Setup Effort|Zero|A lot|A little|
|Coverage|~80%|~99%|100%|
|Fail means|Code fault|Code fault|Code? Infra? Other services?|

I don't think there's a superior test suit. They should be combined. What we must determine is what and why one of them comes first.



![image](https://user-images.githubusercontent.com/12825679/104560112-bceb2480-5688-11eb-882d-c8312ffb8a1a.png)


For Computation, here's my project. Indexer. It scrapes data from DB and does some logic things. After that it sends to Elasticsearch. Scoring is one of business logics here and what it does is computation job. So how we can test it? Since computation is a pure function, we can test with unit test. 






![image](https://user-images.githubusercontent.com/12825679/104560778-ba3cff00-5689-11eb-9ba1-ea2d25a85ea7.png)


In my opinion, we could have three types of testing. Unit Testing for Computation features, Full Stack Testing for CRUD features and lastly Monitoring for vary environments.

All I want from my experience is that don't apply Unit Testing for CRUD features. Futhermore it also helps how we architecture well. (Just like Command and Query Model)

# References

- [https://testing.googleblog.com/2015/04/just-say-no-to-more-end-to-end-tests.html](https://testing.googleblog.com/2015/04/just-say-no-to-more-end-to-end-tests.html)
- [https://docs.microsoft.com/en-us/azure/devops/learn/devops-at-microsoft/evolving-test-practices-microsoft](https://docs.microsoft.com/en-us/azure/devops/learn/devops-at-microsoft/evolving-test-practices-microsoft)

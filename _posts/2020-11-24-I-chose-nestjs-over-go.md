---
layout: post
title: I chose Nest.js over Golang
author: Eunchan Lee
---

<img width=500 src="https://i.imgur.com/ukMIHvR.png"/>
<img width=500 src="https://i.imgur.com/5lWbwaC.png"/>

### `Nest.js` has rich features.

GraphQL, WebScoket, Queue(Redis), MQTT, gRPC, Kafka, OpenAPI, etc...

These are **built-in** features.

Of course there is a good framework like **Beego** but above things **cannot be found** there.


### `Nest.js` has a structure.

When I need to add a feature, there are some assured places where the files have to go.

Do you find it uncomfortable?

Without it, you need to build your **own architecture** which ends up being quite **similar one** from `Nest.js`.

Unless you are developing something like **dedicated super-optimized web server**, your business logic, I'm quite sure, will seems like `Nest.js` soon.

Five projects from three coworkers of mine had been developed on their own way. 

Guess what? They all have Services, Controllers and Models in resemble directories. And its composition also **resembles each other, seriously.**

Guys, are you sure you're going to build very special things?


### `Nest.js` is not bad for serverless.

I mean **cold start**.

To be honest, most of languages/frameworks have no problem for serving serverlessly, **except JVM** 😅

### `Nest.js` is not slow.

```ts
@Get(":id")
async read(@Param("id") id: number): Promise<Event> {
    const e = await this.eventSvc.findOne(id);
    e.visit++;
    e.updatedAt = new Date();
    // empty loop
    for (let i = 0; i < 1000000; i++) {
        i = i;
    };
    return this.eventSvc.save(e);
}
```
```go
func (h *Handler) handleReadEvent(c echo.Context) error {
	idStr := c.Param("id")
	idStr = strings.TrimSpace(idStr)
	id, err := strconv.Atoi(idStr)
	if err != nil {
		return c.NoContent(400)
	}

	e := Event{}
	res := h.db.First(&e, id)
	if errors.Is(res.Error, gorm.ErrRecordNotFound) {
		return c.NoContent(404)
	}

	e.Visit++
	e.UpdatedAt = time.Now()
	// empty loop
	for i := 0; i < 1000000; i++ {
		i = i
	}
	res = h.db.Save(&e)
	if res.Error != nil {
		err := fmt.Errorf("error while updating %v : %w", e.Id, res.Error)
		fmt.Println(err)
	}

	return c.JSON(200, e)
}
```

|Name|P50|P90|P99|
|-:|-:|-:|-:|
|Nest.js|84ms|109ms|212ms|
|Echo|78ms|121ms|151ms|
|Diff|+6ms|-12ms|+61ms|
|Diff(Pct.)|+7.6%|-10%|+40%|

Apparently, not fast neither.

Remember that I said **not slow**. 😅

### `Golang` is not for a big web server.

To me it's more like **Blazing Fast Strong Type Script with Binary Compilation**.

That's why I made a search indexer and kubernetes tools with Golang.

### `Golang` has Beego, Buffalo, etc.

Hmm. I have tried to use them but you know that maintance and communities.

Buffalo seems not active anymore.

Beego has fewer features.

To serve a JSON in Beego, you should write like:
```go
func (this *AddController) Get() {
    mystruct := { ... }
    this.Data["json"] = &mystruct
    this.ServeJSON()
}
```

If you're happy with that, congrats!

I think I'm quite picky 🤨

### `Golang` has no generic. not yet tho.

Generic is the most productive language feature I think.

Go hasn't so Go isn't.

I've heard it's in developing. I yarn for it.

### To sum up

I prefer using **Nest.js** to **any libraries in golang** for canonical web servers.

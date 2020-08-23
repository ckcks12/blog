---
layout: post
title: Go vs. Java vs. Kotlin-Native Performance Test
author: Eunchan Lee
tags: [programming,go,java,kotlin,performance]
---

# Source
### main.go
```go
package main

func main() {
   s := 0
   for i := 0; i < 2147483647; i++ {
       s += i
       s -= i
   }
   println(s)
}
```
### main.kt
```kt
fun main() {
   var s = 0
   for (i in 0..2147483646) {
       s += i
       s -= i
   }
   println(s)
}
```

# Result
|Language|Compile|Execution|File Size|JVM Required|
|:-|-:|-:|-:|:-:|
|Go|0.11s|0.55s|1.1M|X|
|Java(by kotlinc)|5.51s|1.15s|691B|O|
|Kotlin Native|12.46s|2.85s|912K|X|

# Why? Why is it **super** slower than others?
Well, I don't know 😅 How do I know that?

Kotlin Native requires XCode CLI. Also compiler downloads a lot of dependencies the first time it runs. 

Instead, Go generates its own binary (with using gcc and stuff).

Maybe, that's the key I guess 🤨

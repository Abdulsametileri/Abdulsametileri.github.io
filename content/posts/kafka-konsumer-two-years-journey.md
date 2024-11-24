---
title: "Kafka Konsumer Two Years Journey"
date: "2024-07-18"
description: "The state of Kafka consumer libraries and new features two years after their release."
summary: "Last year, we introduced kafka-konsumer and kafka-cronsumer libraries in their articles. After one year of development, we are thrilled to write a new blog post introducing other useful features and our use cases. During those two years of journey, these two libraries have been used in 100+ projects in Trendyol. 🚀"
tags: [ "kafka", "go" ]
categories: [ "kafka", "go" ]
---

Last year, we introduced [kafka-konsumer](https://medium.com/trendyol-tech/kafka-konsumer-c47b4b8c1599) and
[kafka-cronsumer](https://medium.com/trendyol-tech/kafka-exception-c-r-onsumer-37c459e4849d) libraries in their
articles. After one year of development,
we are thrilled to write a new blog post introducing other useful features and our use cases. During those two years of
journey, these two libraries have been used in **100+ projects** in Trendyol. 🚀

First of all, let’s remember what these libraries are. Basically,

- [Kafka Konsumer](https://github.com/Trendyol/kafka-konsumer): ensures the easy implementation of Kafka consumer with a
  built-in exception manager (Kafka-cronsumer).
- [Kafka Cronsumer](https://github.com/Trendyol/kafka-cronsumer): cron-based Kafka exception consumer with the power of
  auto retry & concurrency

{{< figure src="/images/kktyj/konsumerlogo.webp" align="center">}}

As a Seller Product Indexing Team, we have used these libraries for every consumer project. It has worked in production
for a very long time; it gave a good performance 💪. For example,

{{< figure src="/images/kktyj/categoryint.webp" align="center"
caption="Figure: Lag Graph, 16 pods and 16 partitions for the international category" >}}

{{< figure src="/images/kktyj/contentfeaturesint.webp" align="center"
caption="Figure: Lag Graph, 16 pods and 16 partitions for the content features" >}}

{{<figure src="/images/kktyj/tpgraph.webp" align="center"
caption="Figure: TP Graph, Lots of different types of messages under the heavy load" >}}

Last year, we also
wrote 
[New Winner of Kafka Consumers: Scala to Go Journey 350 Million Messages per Day](https://abdulsamet-ileri.medium.com/new-winner-of-kafka-consumers-scala-to-go-journey-604c6bdd7041).
You can find our re-platforming journey and more performance insights!

Let’s take a look at what we develop and our use cases.👀

---
# Apicurio Registry Serdes v2系でKafkaのメッセージをシリアライズする

Kafkaのメッセージをスキーマ言語でシリアライズ/デシリアライズするためには、
専用のライブラリ(Serdes)が必要になります。
Serdesの1つであるApicurio Registry Serdesを使っていたら、
v1系とv2系でパッケージ名など大きく変わっていてハマったので使い方をメモします。

ググった範囲だとまだv1系を使った記事がヒットするので、これからv2系を使う方は参考になるはずです。


## 何が変わったのか

Apicurio Regisry Serdesをv1系とv2系で比較した結果は以下の通りです。

|項目|v1|v2|
|:--|:--|:--|
|groupId|io.apicurio|io.apicurio|
|artifactId|apicurio-registry-utils-serde|apicurio-registry-serdes-<スキーマ言語名>-serde|
|Serializer/Deserializerのパッケージ名|io.apicurio.registry.utils.serde|io.apicurio.registry.serde.<スキーマ言語名>|
|Serializerのクラス名(Apache Avroの場合)|AvroKafkaSerializer|AvroKafkaSerializer|

v2系の<スキーマ言語名>の部分には、`avro`(Apache Avro)、`protobuf`(Google Protocol Buffers)、`jsonschema`(JSON Schema)が入ります。

大きな違いとしては、ライブラリ名(`artifactId`)とパッケージ名が別物になっています。
v1系では1つのSerdesに複数のスキーマ言語用のクラスが用意されていましたが、
v2系からはスキーマ言語ごとにSerdesが分割されたようです。
メッセージをシリアライズ/デシリアライズするクラス名は変更なしです。

Apicurio Registryのメジャーバージョンに合わせてSerdesのバージョンをv1系からv2系にしたい場合もあると思います。しかし、Maven Centralにはv1系のライブラリ名でv2系で検索しても見つからないので注意が必要です。


## v2系のSerdesを使ったサンプル

変更点が分かったところで、サンプルコードです。
Apicurio Registry Serdes v2とGradle、Kotlin、スキーマ言語にはApache Avroを使ってみます。

まずはGradleにSerdes v2系を依存関係に含めます。
この記事を書いた時点(2021/06/23)のバージョンは `2.0.1.Final` です。

**build.gradle.kts**
```kotlin
import org.jetbrains.kotlin.gradle.tasks.KotlinCompile

plugins {
    java
    kotlin("jvm") version "1.5.10"
    id("com.github.davidmc24.gradle.plugin.avro") version "1.2.0"
}

java {
    sourceCompatibility = JavaVersion.VERSION_11
    targetCompatibility = JavaVersion.VERSION_11
}

group = "com.example"
version = "1.0-SNAPSHOT"

repositories {
    mavenCentral()
    gradlePluginPortal()
}

val kafkaVersion = "2.8.0"
val serializerVersion = "2.0.1.Final"
val avroVersion = "1.10.2"
val slf4jVersion = "1.7.30"

dependencies {
    implementation("org.apache.kafka:kafka-clients:$kafkaVersion")
    implementation("io.apicurio:apicurio-registry-serdes-avro-serde:$serializerVersion")
    implementation("org.apache.avro:avro:$avroVersion")
    implementation("org.slf4j:slf4j-simple:$slf4jVersion")
}

tasks.withType<KotlinCompile>() {
    kotlinOptions.jvmTarget = "11"
}
```

シリアライズするメッセージ、Avroでスキーマ定義します。
これを `src/main/avro` ディレクトリに配置して、 `gradle generateAvroJava` を実行すればスキーマに沿ったクラスが生成されます。

**movie.avsc**
```json
{
  "namespace": "com.example",
  "type": "record",
  "name": "Movie",
  "fields": [
    {
      "name": "title",
      "type": "string"
    },
    {
      "name": "year",
      "type": "int"
    }
  ]
```

あとはメッセージをシリアライズして送信するKafka Producerを作成するだけです。今回はここで指定するSerializerのパッケージ名が重要です。

メッセージのシリアライズに利用するSerializerは、`io.apicurio.registry.serde.avro.AvroKafkaSerializer`を使用します。

**src/main/kotlin/AvroProducer.kt**
```kotlin
package com.example

import io.apicurio.registry.serde.avro.AvroKafkaSerializer
import org.apache.kafka.clients.producer.KafkaProducer
import org.apache.kafka.clients.producer.ProducerConfig
import org.apache.kafka.clients.producer.ProducerRecord
import org.apache.kafka.clients.producer.RecordMetadata
import org.apache.kafka.common.serialization.StringSerializer
import io.apicurio.registry.serde.SerdeConfig
import java.util.*

fun main() {
    // create producer properties
    val props = Properties()
    props[ProducerConfig.BOOTSTRAP_SERVERS_CONFIG] = "localhost:9092"
    props[ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG] = StringSerializer::class.qualifiedName
    props[ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG] = AvroKafkaSerializer::class.qualifiedName
    props[SerdeConfig.REGISTRY_URL] = "http://localhost:8080/apis/registry/v2"
    props[SerdeConfig.AUTO_REGISTER_ARTIFACT] = "true"

    // build kafka producer
    val topic = "sample-topic"
    KafkaProducer<String, Movie>(props).use { producer ->

      // build message
      val key = "sample-key-111"
      val serializedValue = Movie.newBuilder()
          .setTitle("Titanic")
          .setYear(1997)
          .build()
      println("Producing record: key=$key, value=$serializedValue")

      // publish message
      producer.send(ProducerRecord(topic, key, serializedValue)) { metadata: RecordMetadata, e: Exception? ->
          when (e) {
              null -> println("Sent record: topic=${metadata.topic()}, partition=${metadata.partition()}, offset=${metadata.offset()}")
              else -> e.printStackTrace()
          }
      }

      producer.flush()
    }
}
```

## 参考

Apicurio Registryのサンプル集にスキーマ言語ごとの実装例があります。

https://github.com/Apicurio/apicurio-registry-examples

---

以上。

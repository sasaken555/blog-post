# MavenでAvroのスキーマファイルからJavaコードを生成するのにつまずいた

Kafka Tutorialsを進めていたときに、タイトル通りMavenでAvroのスキーマからJavaのコード生成するのにつまずいた話です。
このサイトに掲載されているサンプルコードはすべてGradleでビルドされているのですが、自分が良く使っているMavenだったらどうかなと実践してみた次第です。

https://kafka-tutorials.confluent.io/

Avroの公式サイトにはMavenのプラグインでAvroのスキーマからJavaのコードを生成する方法が載っています。
Mavenを使う場合、 `mvn compile` を実行するようにガイドされています。

https://avro.apache.org/docs/current/gettingstartedjava.html

しかし、 `mvn compile` をいざ実行してみると、
コンパイルするものがないとメッセージが返ってきます。無慈悲です。

```bash
[INFO] --- maven-compiler-plugin:3.8.0:compile (default-compile) @ kstreams-serialization ---
[INFO] Nothing to compile - all classes are up to date
```

最終的にはStackoverflowの以下のissueにたどり着き、 `mvn avro:schema` を実行すればできると。
よく考えたらgoalsに書いているんだから当然でした。

https://stackoverflow.com/questions/31921464/unable-to-invoke-avro-maven-plugin

`movie.avsc` というスキーマファイルを作成して実際に上記のコマンドを実行してみると、無事 `generated-sources` 配下にJavaのコード `Movie.java` が出力されました。

めでたし、めでたし。

---

しかし、なぜ mvn compileで通らないのかは理解しなければ。

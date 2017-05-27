## **Abstract**

Mobile platform plans to migrate our JSON serialization tool from [Gson](https://github.com/google/gson) to [Moshi](https://github.com/square/moshi). This will start in generated realtime models where they can be safely/easily experimented before taking a larger plunge. Moshi gives us a wealth of improvements over Gson, specifically with regards to API, performance, integration with our existing stack, and longevity.

## **Justification**

### Better API

Moshi has a much improved API over Gson that allows us to do a lot of neat things. In typical Square libraries fashion: it is simple, practical, flexible, and extremely powerful.

* Annotations are a first class part of how Moshi works, allowing you to put information into them for more fine grained/contextual deserialization. Some notable examples:

    * Retrofit turns over these annotations to you as well, allowing us to decorate retrofit services with them. This opens up a lot of doors, and there are projects leveraging this for [RPC semantics](https://github.com/segmentio/retrofit-jsonrpc) and [GraphQL](https://github.com/apollographql/apollo-android) support.

    * Dagger-style qualifiers (@ColorInt @FromHexColor int color)

    * Contextual deserialization - this is a huge step forward, and not something I can easily do justice to covering here. Instead, I’ll link a couple places with neat examples and some uber use cases:

        * [moshi-lazy-adapters](https://github.com/serj-lotutovici/moshi-lazy-adapters#list-of-provided-adapters) 

        * [Moshi’s recipes](https://github.com/square/moshi/tree/master/examples/src/main/java/com/squareup/moshi/recipes)

* [Better, more human readable serialization failure messages.](https://github.com/square/moshi#fails-gracefully)

* newBuilder() API for easy creation of local instances that inherit from upstream adapters

    * Similar to Square’s pattern with OkHttp, Retrofit, etc

    * This opens the door for us to do some very cool things with generated services, such as scoping adapters down alongside RIBs and keeping a minimal amount of them in memory, rather than one beefy instance in memory that understands all 1000+ models.

* Default values

    * This is actually mentioned in the recipes link above, but deserves special emphasis as this could have big implications for us in service generation. With gson, custom adapters are tedious to maintain and tend to have to be uniquely generated per target. [This example](https://github.com/square/moshi/blob/master/examples/src/main/java/com/squareup/moshi/recipes/DefaultOnDataMismatchAdapter.java) demonstrates a concise, reusable implementation leveraging Moshi’s JsonValue API.

* Less [footguns](http://www.urbandictionary.com/define.php?term=footgun) around null safety and handling of custom classes like Date and UUID as compared to Gson.

    * Gson has a default Date format. Your guess to what it does is as good as anyone’s. Timezones are out the window, especially if the encoder and decoder are in, uh, *different* timezones.

    * Gson does similarly odd things with UUID, which are beholden to the JDK implementation details.

    * Moshi has a neat failOnUnknown() feature that lets you explode on unrecognized types. This has a few possible use cases, such as integration testing against the payloads server is sending.

* [Square Eng blog post](https://medium.com/square-corner-blog/moshi-another-json-processor-624f8741f703)

### Performance

Moshi is faster than Gson and more memory performant.

* [Benchmarks](https://github.com/hzsweers/json-serialization-benchmarking)

* How?

    * Moshi uses [Okio](https://github.com/square/okio) under the hood. Aside from being a very neat little IO library, it is also the same one that powers our network and local storage stack. What this means is that we could see some nice wins in memory consumption during deserialization because retrofit will facilitate allowing [OkHttp and Moshi to share Okio buffer segments under the hood during deserialization](https://github.com/square/retrofit/blob/master/retrofit-converters/moshi/src/main/java/retrofit2/converter/moshi/MoshiResponseBodyConverter.java#L35-L45), whereas currently they use totally separate memory buffers for holding their data. 

    * Moshi leverages Okio’s select() API. In short, the select() API allows you to tell Okio what strings you "expect" ahead of time (in the form of an array). Then, you can prod Okio to read the next string in the context of that array, and it will just give you back an index into that array without reading the entire string (because it doesn’t need to!) and then potentially throwing it away. In JSON serialization, we *do* know our keys up front, so we can leverage this. A bridge to this API was opened up in the most recent release of Moshi, and auto-value-moshi [now leverages it](https://github.com/rharter/auto-value-moshi/pull/76). This is what gives Moshi the speed win over Gson.

## Other considerations

* Gson *is not being actively developed anymore, or maintained by Google*. This is a key consideration. Two of the three maintainers of Gson are the creators of Moshi (Jesse Wilson and Jake Wharton), and they advocate for use of Moshi now. In a way, it’s the Gson 3.0 that wasn’t ever going to happen.

* Moshi has a substantially smaller footprint (122kB as of v1.4) compared to Gson (~300kB).

* Okio is also what we use in the presidio storage framework for disk IO. Should we ever decide to move to JSON for local storage (for parity with network models), Moshi would integrate quite nicely with similar performance benefits from shared buffer usage.

* What about switching to a binary format?

    * This may happen some day, but we are not likely to completely move away from JSON as a data format for some time. In the meantime, we’re going to improve our tooling.

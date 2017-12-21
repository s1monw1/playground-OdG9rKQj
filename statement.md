# Kotlin Web Applications with Ktor

_Disclaimer: My articles are published under 
<a href="https://creativecommons.org/licenses/by-nc-nd/4.0/legalcode" target="_blank">"Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0)"</a>._

© Copyright: Simon Wirtz, 2017

This article is featured in the DZone [Guide to Web Development](https://dzone.com/guides/web-development-frameworks-and-responsive-design). Get your free copy for more insightful articles, industry statistics, and more!

Feel free to share.

## Introduction
When Google made **Kotlin** an *official language* for Android a few months ago at [Google I/O](https://twitter.com/Android/status/864911929143197696), the language gained a lot of popularity in the Android world quickly. On the server side though, Kotlin is not as broadly adopted yet and some people still seem to be cautious when backend services are involved. Other developers are convinced that Kotlin is mature enough and can safely be used for any server application in which Java could play a role otherwise.

If you want to develop web apps with Kotlin, you can choose from various web frameworks like Spring MVC/WebFlux, Vert.x, Vaadin and basically everything available for the JVM. Besides the mentioned frameworks there's also _a Kotlin specific_ library available for creating web applications, called [ktor](https://github.com/ktorio/ktor). After reading this article, you'll know what ktor and its advantages are and how you can quickly develop a web application in Kotlin.

## Ktor

The web application framework **ktor**, itself written in Kotlin, is meant to provide a tool for quickly creating web applications with Kotlin. The resulting software may be hosted in common servlet containers like Tomcat or standalone in a [Netty](https://netty.io) for example. Whatever kind of hosting you choose, ktor is making heavy use of Kotlin Coroutines, so that it's implemented 100% asynchronously and mainly non-blocking. ktor does not dictate which frameworks or tools to be used, so that you can choose whatever logging, DI, or templating engine you like. The library is pretty light-weight in general, still being very extensible through a plugin mechanism.

One of the greatest advantages attributed to Kotlin is its ability to provide type-safe builders, also known as Domain Specific Languages ([DSL](https://blog.simon-wirtz.de/creating-dsl-with-kotlin-introducing-a-tlslibrary/)). Many libraries already provide DSLs as an alternative to common APIs, such as the Android lib [anko](https://github.com/Kotlin/anko), the Kotlin [html](https://github.com/Kotlin/kotlinx.html) builder or the freshly released [Spring 5](https://blog.simon-wirtz.de/spring-webflux-with-kotlin-reactive-web/) framework. As we will see in the upcoming examples, ktor also makes use of such DSLs, which enable the user to define the web app's endpoints in a very declarative way.

## Example Application

In the following, a small RESTful (Not sure, if everyone will agree) web service will be developed with **ktor**. The service will use an in-memory repository with simple `Person` resources, which it exposes through a JSON API. Let's look at the components.

### `Person` Resource and Repository
According to the app's requirements, a resource "`Person`" is defined as a `data class` and the corresponding repository as an `object`, Kotlin's way of applying the Singleton pattern to a class.

~~~ kotlin
data class Person(val name: String, val age: Int){
    var id: Int? = null
}
~~~

The resource has two simple properties, which need to be defined when a new object is constructed, whereas the `id` property is set later when stored in the repository.

The `Person` **repository** is rather unspectacular and not worth observing. It uses an internal data store and provides common CRUD operations.

### Endpoints
The most important part of the web application is the configuration of its endpoints, exposed as a REST API. The following endpoints will be implemented:

|	Endpoint	| HTTP Method	| Description	|
| -------------------	|-----------------| --------------------	|
|	**/persons**	|	GET | Requests all `Person` resources	|
|	**/persons/{id}**	|	GET | Requests a specific `Person` resource by its ID	|
|	**/persons**		|	DELETE |Requests all `Person` resources to be removed	|
|	**/persons/{id}**	|	DELETE |Requests a specific `Person` resource to be removed by its ID	|
|	**/persons**	|	POST |Requests to store a new `Person` resource|
|	**/**	|	GET | Delivers a simple HTML page welcoming the client	|

The application won't support an update operator via `PUT`.

### Routing with Ktor

Now ktor comes into play with its structured DSL, which will be used for defining the previously shown endpoints, a process often referred to as "routing". Let's see how it works:

~~~ kotlin
fun Application.main() {
    install(DefaultHeaders)
    install(CORS) {
        maxAge = Duration.ofDays(1)
    }
    install(ContentNegotiation){
        register(ContentType.Application.Json, GsonConverter())
    }

    routing {
        get("/persons") {
            LOG.debug("Get all Person entities")
            call.respond(PersonRepo.getAll())
        }
    }
    // more routings
}

~~~

The fundamental class in the ktor library is `Application`, which represents a configured and eventually running web service instance. In the snippet, an extension function `main()` is defined on `Application`, in which it's possible to call functions defined in `Application` directly, without additional qualifiers. This is done by invoking `install()` multiple times, a function for adding certain `ApplicationFeature`s into the request processing pipeline of the application. These features are optional and the user can choose from various types. The only interesting feature in the shown example is `ContentNegotiation`, which is used for the (de-)serialization of Kotlin objects to and from JSON, respectively. The Gson library is used as a backing technology.

The routing itself is demonstrated by defining the `GET` endpoint for retrieving all `Person` resources here. It's actually straight-forward since the result of the repository call `getAll()` is just delegated back to the client. The `call.respond()` invocation will eventually reach the `GsonSupport` feature taking care of transforming the `Person` into its JSON representation. The same happens for the remaining four REST endpoints, which will not be shown explicitly here.

#### The HTML builder

Another cool feature of ktor is its integration with other Kotlin DSLs like the HTML builder, which can be found on [GitHub](https://github.com/Kotlin/kotlinx.html). By adding a single additional dependency to a ktor app, this builder can be used with the extension function `call.respondHtml()`, which expects type-safe HTML code to be provided. In the following example, a simple greeting to the readers is included, which the web service exposes as its index page.

~~~ kotlin

get("/") {
    call.respondHtml {
        head {
            title("ktor Example Application")
        }
        body {
            h1 { +"Hello DZone Readers" }
            p {
                +"How are you doing?"
            }
        }
    }
}
~~~

### Starting the Application

After having done the configuration of a ktor application, there's still a need to start it through a regular `main` method. In the demo, a [Netty](https://netty.io) server is started standalone by doing the following:

~~~ kotlin
fun main(args: Array<String>) {
    embeddedServer(Netty, 8080, module = Application::main).start(wait = true)
}
~~~

The starting is made pretty simple by just calling the `embeddedServer()` method with a Netty environment, a port and the module to be started, which happens to be the thingy defined in the `Application.main()` extension function from the previously shown example.

### Testing 

A web framework or library is only applicable if it comes with integrated testing means, which ktor actually does. Since the `GET` routing for retrieving all `Person` resources was already shown, let's have a look at how the endpoint can be tested.

~~~ kotlin
  @Test
    fun getAllPersonsTest() = withTestApplication(Application::main) {
        val person = savePerson(gson.toJson(Person("Bert", 40)))
        val person2 = savePerson(gson.toJson(Person("Alice", 25)))
        handleRequest(HttpMethod.Get, "/persons") {
            addHeader("Accept", json)
        }.response.let {
            assertEquals(HttpStatusCode.OK, it.status())
            val response = gson.fromJson(it.content, Array<Person>::class.java)
            response.forEach { println(it) }
            response.find { it.name == person.name } ?: fail()
            response.find { it.name == person2.name } ?: fail()
        }
        assertEquals(2, PersonRepo.getAll().size)
    }
~~~

This one is quite simple: Two resource objects are added to the repository, then the `GET` request is executed and some assertions are done against the web service response. The important part is `withTestApplication()`, which ktor offers through its `testing` module and makes it possible to directly test the `Application`. 

The article presented basically everything worth knowing in order to get started with ktor web applications. For more details, I recommend the [ktor](http://ktor.io) homepage or the [samples](https://github.com/Kotlin/ktor/tree/master/ktor-samples) included in the ktor repository.

## Takeaways

In this article we had a look at the Kotlin web service library **ktor** that provides simple tools for writing a light-weight web application in Kotlin very quickly. ktor makes heavy use of Kotlin's beautiful features, especially DSLs, Coroutines and extension functions, all of which make the language and ktor itself so elegant. This mainly separates ktor from other powerful frameworks, which in fact can be used with Kotlin very smoothly, but, since not completely written in Kotlin, lack certain features you can find in ktor. It's a great place to start with web applications because of its simplicity and ability to be extended with custom features if needed, following the principle “simple is easy, complex is available”.

The complete source code of the developed web application can be found on [GitHub](https://github.com/s1monw1/ktor_application), it's backed by a Gradle build.

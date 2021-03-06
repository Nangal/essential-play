## Routes in Depth

The previous section introduced `Actions`, `Controllers`, and routes.
`Actions` and `Controllers` are standard Scala code,
but routes are something new and specific to Play.

We define Play routes using a special DSL that compiles to Scala code.
The DSL provides both a convenient way of mapping URIs to method calls
and a way of mapping method calls *back* to URIs.
In this section we will take a deeper look at Play's routing DSL
including the various ways we can extract parameters from URIs.

### Path Parameters

Routes associate *URI patterns* with *action-producing method calls*.
We can specify *parameters* to extract from the URI and pass to our controllers.
Here are some examples:

~~~ coffee
# Fixed route (no parameters):
GET /hello controllers.HelloController.hello

# Single parameter:
GET /hello/:name controllers.HelloController.helloTo(name: String)

# Multiple parameters:
GET /send/:msg/to/:user ↩
  controllers.ChatController.send(msg: String, user: String)

# Rest-style parameter:
GET /download/*filename ↩
  controllers.DownloadController.file(filename: String)
~~~

The first example assocates a single URI with a parameterless method.
The match must be exact---only `GET` requests to `/hello` will be routed.
Even a trailing slash in the URI (`/hello/`) will cause a mismatch.

The second example introduces a *single-segment parameter*
written using a leading colon (':').
Single-segment parameters match any continuous set of characters
*excluding* forward slashes ('/').
The parameter is extracted and passed
to the method call---the rest of the URI must match exactly.

The third example uses two single-segment parameters
to extract two parts of the URI.
Again, the rest of the URI must match exactly.

The final example uses a *rest-parameter*
written using a leading asterisk ('*').
Rest-style parameters match all remaining characters in the URI,
including forward slashes.

### Matching Requests to Routes

When a request comes in, Play attempts to route it to an action.
It examines each route in turn until it finds a match.
If no routes match, it returns a 404 response.

Routes match if the HTTP method has the relevant value
and the URI matches the shape of the pattern.
Play supports all eight HTTP methods:
`OPTIONS`, `GET`, `HEAD`, `POST`, `PUT`, `DELETE`, `TRACE`, and `CONNECT`.

:Routing examples---mappings from HTTP data to Scala code

---------------------------------------------------------------------------------------------
HTTP method and URI                Scala method call or result
---------------------------------- ----------------------------------------------------------
`GET  /hello`                      `controllers.HelloController.hello`

`GET  /hello/dave`                 `controllers.HelloController.helloTo("dave")`

`GET  /send/hello/to/dave`         `controllers.ChatController.send("hello", "dave")`

`GET  /download/path/to/file.txt`  `controllers.DownloadController.file("path/to/file.txt")`

`GET  /hello`/                     404 result (trailing slash)

`POST /hello`                      404 result (POST request)

`GET  /send/to/dave`               404 result (missing path segment)

`GET  /send/a/message/to/dave`     404 result (extra path segment)
---------------------------------------------------------------------------------------------

<div class="callout callout-info">
*Play Routing is Strict*

Play's strict adherance to its routing rules can sometimes be problematic.
Failing to match the URI `/hello/`, for example, may seem overzealous.
We can work around this issue easily
by mapping multiple routes to a single method call:

~~~ coffee
GET  /hello  controllers.HelloController.hello # no trailing slash
GET  /hello/ controllers.HelloController.hello # trailing slash
POST /hello/ controllers.HelloController.hello # POST request
# and so on...
~~~
</div>

### Query Parameters

We can specify parameters in the method-call section
of a route without declaring them in the URI.
When we do this Play extracts the values from the query string instead:

~~~ coffee
# Extract `username` and `message` from the path:
GET /send/:message/to/:username ↩
  controllers.ChatController.send(message: String, username: String)

# Extract `username` and `message` from the query string:
GET /send ↩
  controllers.ChatController.send(message: String, username: String)

# Extract `username` from the path and `message` from the query string:
GET /send/to/:username ↩
  controllers.ChatController.send(message: String, username: String)
~~~

We sometimes want to make query string parameters optional.
To do this, we just have to define them as `Option` types.
Play will pass `Some(value)` if the URI contains the parameter
and `None` if it does not.

For example, if we have the following `Action`:

~~~ scala
object NotificationController {
  def notify(username: String, message: Option[String]) =
    Action { request => /* ... */ }
}
~~~

we can invoke it with the following route:

~~~ coffee
GET /notify controllers.NotificationController. ↩
  notify(username: String, message: Option[String])
~~~

We can mix and match required and optional query parameters as we see fit.
In the example, `username` is required and `message` is optional.
However, *path* parameters are always required---the following route
fails to compile because the path parameter `:message` cannot be optional:

~~~ coffee
GET /notify/:username/:message controllers.NotificationController. ↩
  notify(username: String, message: Option[String])

# Fails to compile with the following error:
#     [error] conf/routes:1: No path binder found for Option[String].
#     Try to implement an implicit PathBindable for this type.
~~~

:Query string parameter examples

--------------------------------------------------------------------------------
HTTP method and URI                      Scala method call or result
---------------------------------------- ---------------------------------------
`GET  /send/hello/to/dave`               `ChatController.send("hello", "dave")`

`GET  /send?message=hello&username=dave` `ChatController.send("hello", "dave")`

`GET  /send/to/dave?message=hello`       `ChatController.send("hello", "dave")`
--------------------------------------------------------------------------------

### Typed Parameters

We can extract path and query parameters of types other than `String`.
This allows us to define `Actions` using well-typed arguments without messy parsing code.
Play has built-in support for `Int`, `Double`, `Long`, `Boolean`, and `UUID` parameters.

For example, given the following route and action definition:

~~~ coffee
GET /say/:msg/:n/times controllers.VerboseController.say(msg: String, n: Int)
~~~

~~~ scala
object VerboseController extends Controller {
  def say(msg: String, n: Int) = Action { request =>
    Ok(List.fill(n)(msg) mkString "\n")
  }
}
~~~

We can send requests to URLs like `/say/Hello/5/times`
and get back appropriate responses:

~~~ bash
bash$ curl -v 'http://localhost:9000/say/Hello/5/times'
# HTTP headers...
Hello
Hello
Hello
Hello
Hello

bash$
~~~

Play also has built-in support for `Option` and `List` parameters in the
query string (but not in the path):

~~~ coffee
GET /option-example controllers.MyController.optionExample(arg: Option[Int])
GET /list-example   controllers.MyController.listExample(arg: List[Int])
~~~

`Optional` parameters can be specified or omitted and `List` parameters can
be specified any number of times:

~~~ coffee
/option-example             # => MyController.optionExample(None)
/option-example?arg=123     # => MyController.optionExample(Some(123))
/list-example               # => MyController.listExample(Nil)
/list-example?arg=123       # => MyController.listExample(List(123))
/list-example?arg=12&arg=34 # => MyController.listExample(List(12, 34))
~~~

If Play cannot extract values of the correct type for each parameter in a route,
it returns a *400 Bad Request* response to the client.
It doesn't consider any other routes lower in the file.
This is standard behaviour for all types of path and query string parameter.

<div class="callout callout-warning">
*Custom Parameter Types*

Play parses route parameters using instances of two different *type classes*:

 - [`play.api.mvc.PathBindable`] to extract path parameters;
 - [`play.api.mvc.QueryStringBindable`] to extract query parameters.

We can implement custom parameter types
by creating implicit values these type classes.
</div>

### Reverse Routing

*Reverse routes* are objects that we can use to generate URIs.
This allows us to create URIs from type-checked program code
without having to concatenate `Strings` by hand.

Play generates reverse routes for us
and places them in a `controllers.routes`
package that we can access from our Scala code.
Returning to our original routes for `HelloController`:

~~~
GET /hello       controllers.HelloController.hello
GET /hello/:name controllers.HelloController.helloTo(name: String)
~~~

The route compiler generates a `controllers.routes.HelloController`
object with reverse routing methods as follows:

~~~ scala
package routes

import play.api.mvc.Call

object HelloController {
  def hello: Call =
    Call("GET", "/hello")

  def helloTo(name: String): Call =
    Call("GET", "/hello/" + encodeURIComponent(name))
}
~~~

We can use reverse routes to reconstruct [`play.api.mvc.Call`] objects
containing the information required to address `hello` and `helloTo` over HTTP:

~~~ scala
import play.api.mvc.Call

val methodAndUri: Call =
  controllers.routes.HelloController.helloTo("dave")

methodAndUri.method // "GET"
methodAndUrl.url    // "/hello/dave"
~~~

Play's HTML form templates, in particular, make use of `Call` objects
when writing HTML for `<form>` tags.
We'll see these in more detail next chapter.

### Take Home Points

*Routes* provide bi-directional mapping between URIs and
`Action`-producing methods within `Controllers`.

We write routes using a Play-specific DSL that compiles to Scala code.
Each route comprises an HTTP method, a URI pattern,
and a corresponding method call.
Patterns can contain *path* and *query parameters*
that are extracted and used in the method call.

We can *type* the path and query parameters in routes
to simplify the parsing code in our controllers and actions.
Play supports many types out of the box,
but we can also write code to map our own types.

Play also generates *reverse routes* that map method calls back to URIs.
These are placed in a synthetic `routes` package
that we can access from our Scala code.

### Exercise: Calculator-as-a-Service

The `chapter2-calc` directory in the exercises contains
an unfinished Play application for performing various mathematical calculations.
This is similar to the last exercise,
but the emphasis is on defining more complex routes.

Complete this application by filling in the missing actions and routes.
Implement the missing actions marked `TODO` in `app/controllers/CalcController.scala`,
and complete `conf/routes` to hook up the specified URLs:

 - `CalcController.add` and `CalcController.and` are examples of `Actions`
   involving typed parameters;

 - `CalcController.concat` is an example involving a rest-parameter;

 - `CalcController.sort` is an example involving a parameter with a
   parameterized type;

 - `CalcController.howToAdd` is an example of reverse routing.

Test your code using `curl` if you're using Linux or OS X
or a browser if you're using Windows:

~~~ bash
bash$ curl 'http://localhost:9000/add/123/to/234'
357

bash$ curl 'http://localhost:9000/and/true/with/true'
true

bash$ curl 'http://localhost:9000/concat/foo/bar/baz'
foobarbaz

bash$ curl 'http://localhost:9000/sort?num=1&num=3&num=2'
1 2 3

bash$ curl 'http://localhost:9000/howto/add/123/to/234'
GET /add/123/to/234
~~~

Answer the following questions when you're done:

1.  What happens when you add a URL-encodeD forward slash (`%2F`)
    to the argument to `concat`? Is this the desired behaviour?

    ~~~ bash
    bash$ curl 'http://localhost:9000/concat/one/thing%2Fthe/other'
    ~~~

    How does the URL-decoding behaviour of Play differ
    for normal parameters and rest-parameters?

2.  Do you need to use the same parameter name in `conf/routes`
    and in your actions? What happens if they are different?

3.  Is it possible to embed a parameter
    of type `List` or `Option`  in the path part of the URL?
    If it is, what do the resulting URLs look like?
    If it is not, what error message do get?

<div class="solution">
As with the previous exercise the `add`, `and`, `concat`, and `sort`
`Actions` simply involve manipulating types to build `Results`:

~~~ scala
def add(a: Int, b: Int) = Action { request =>
  Ok((a + b).toString)
}

def and(a: Boolean, b: Boolean) = Action { request =>
  Ok((a && b).toString)
}

def concat(args: String) = Action { request =>
  Ok(args.split("/").map(decode).mkString)
}

def sort(numbers: List[Int]) = Action { request =>
  Ok(numbers.sorted mkString " ")
}
~~~

`howToAdd` is more interesting. We can avoid hard-coding
the URL for `add` by using its reverse route:

~~~ scala
def howToAdd(a: Int, b: Int) = Action { request =>
  val call = routes.CalcController.add(a, b)
  Ok(call.method + " " + call.url)
}
~~~

The `routes` file is straightforward if you follow the examples above:

~~~ coffee
GET /add/:a/to/:b       controllers.CalcController.add(a: Int, b: Int)
GET /and/:a/with/:b     controllers.CalcController.and(a: Boolean, b: Boolean)
GET /concat/*args       controllers.CalcController.concat(args: String)
GET /sort               controllers.CalcController.sort(num: List[Int])
GET /howto/add/:a/to/:b controllers.CalcController.howToAdd(a: Int, b: Int)
~~~

The answers to the questions are as follows:

1.  If we pass a `%2F` to the route here,
    we end up with the same undesirable `%2F` in the result.

    This happens because `args` is a rest-parameter.
    Play treats rest-parameters differently from
    regular path and query string parameters.

    Because regular parameters are always a single path segment,
    we know there will never be a reserved URL character
    such as a `/`, `?`, `&` or `=` in the content.
    Play is able to reliably decode any URL encoded characters
    for us without fear of ambiguity,
    and does so automatically before calling our `Action`.

    Rest-parameters, on the other hand, can contain unencoded `/` characters.
    Play cannot decode the content without causing ambiguity
    so it passes the raw string captured from the URL without decoding.

    To correctly handle URL encoded characters,
    we have to split the rest parameter on instances of `/`
    and apply the `urlDecode` function to each segment:

    ~~~ scala
    args.split("/").map(urlDecode)
    ~~~

    In example in the question, the controller should remove the `/`
    characters from the parameter and decode the `%2F`,
    yielding a response of `onething/theother`.

2.  Play matches parameters in routes by position rather than by name,
    so we don't have to use the same names in our routes and our controllers.

    In certain circumstances this behaviour can be useful.
    In `sort`, for example, we want a singular parameter name in the URL:

    ~~~ bash
    curl 'http://localhost:9000/sort?num=1&num=3&num=2'
    ~~~

    and a plural name in the action:

    ~~~ scala
    def sort(numbers: List[Int]) = ???
    ~~~

    This can beome confusing when using named arguments on reverse routes.
    Reverse routes take their parameter names from the `conf/routes` file,
    *not* from our `Actions`.
    Calls to the action and the reverse route may therefore look different:

    ~~~ scala
    // Direct call to the Action:
    controllers.CalcController.sort(numbers = List(1, 3, 2))

    // Call to the reverse route:
    routes.CalcController.sort(num = List(1, 3, 2))
    ~~~

3.  Play uses two different type classes
    for encoding and decoding URL parameters:
    `PathBindable` for path parameters
    and `QueryStringBindable` for query string parameters.

    Play provides default implementations of `QueryStringBindable`
    for `Optional` and `List` parameters,
    but it doesn't provide `PathBindables`.

    If we attempt to create a path parameter of type `List[...]`:

    ~~~ coffee
    # We've added `:num` to the `sort` route from the solution
    # to change the required type class from QueryStringBindable to PathBindable:
    GET /sort/:num controllers.CalcController.sort(num: List[Int])
    ~~~

    we get a compile error because of the failure to find a `PathBindable`:

    ~~~ bash
    [error] /Users/dave/dev/projects/essential-play-code/ ↩
            chapter2-calc/conf/routes:4: ↩
            No URL path binder found for type List[Int]. ↩
            Try to implement an implicit PathBindable for this type.
    ~~~
</div>

Now we have seen what we can do with routes,
let's look at the code we can write to handle
`Request` and `Result` objects in our applications.
This will arm us with all the knowledge we need
to start working with HTML and forms in the next chapter.

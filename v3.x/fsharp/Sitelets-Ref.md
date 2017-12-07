# Sitelets Inferred URL Cheat Sheet

Here is a quick reference for the requests parsed by the `Sitelet.Infer` family of functions, and the URLs generated by the corresponding `Context.Link`.

## Base types

```fsharp
type EndPoint = string

(* Accepted Request:    GET /home

   Parsed endpoint: *)  "home"

type EndPoint = int

(* Accepted Request:    GET /2

   Parsed endpoint: *)  2

type EndPoint = float

(* Accepted Request:    GET /1.2345

   Parsed endpoint: *)  1.2345
```

And every standard .NET number type: `int8`, `uint8`, `int16`, `uint16`, `int32`, `uint32`, `int64`, `uint64`, `single`, `double`, `decimal`.

## Records

Encoded as successive path fragments.

```fsharp
type EndPoint =
  { x: string; y: int }

(* Accepted Request:    GET /test/1

   Parsed endpoint: *)  { x = "test"; y = 1 }

type EndPoint =
  { x: string; y: Sub }

and Sub =
  { z: int; t: int }

(* Accepted Request:    GET /test/1/2

   Parsed endpoint: *)  { x = "test"; y = { z = 1; t = 2 } }
```

## Unions

Encoded as an identifying path fragment, followed by the arguments.

```fsharp
type EndPoint =
  | Home
  | Article of id: int * slug: string

(* Accepted Request:    GET /Home

   Parsed endpoint: *)  Home

(* Accepted Request:    GET /Article/1/my-article

   Parsed endpoint: *)  Article(1, "my-article")
```

Choosing the identifying path fragment with `[<EndPoint(fragment)>]`.

```fsharp
type EndPoint =
  | [<EndPoint "/">] Home
  | [<EndPoint "/article">]
    Article of id: int * slug: string

(* Accepted Request:    GET /

   Parsed endpoint: *)  Home

(* Accepted Request:    GET /article/1/my-article

   Parsed endpoint: *)  Article(1, "my-article")
```

When parsing, if several cases have the same `[<EndPoint(fragment)>]`, they are tried in the order in which they are declared:

```fsharp
type EndPoint =
  | [<EndPoint "/blog">]
    AllArticles
  | [<EndPoint "/blog">]
    ArticleById of id: int
  | [<EndPoint "/blog">]
    ArticleBySlug of slug: string

(* Accepted Request:    GET /blog

   Parsed endpoint: *)  AllArticles

(* Accepted Request:    GET /blog/123

   Parsed endpoint: *)  ArticleById 123

(* Accepted Request:    GET /blog/my-article

   Parsed endpoint: *)  ArticleById "my-article"
```

## Datetime

Default format is `"yyyy-MM-dd-HH.mm.ss"`.

```fsharp
type EndPoint = System.DateTime

(* Accepted Request:    GET /2017-05-24-11.41.34

   Parsed endpoint: *)  System.DateTime(2017, 5, 24, 11, 41, 34,
                            System.DateTimeKind.Utc)
```

Setting the format of a DateTime union case argument with `[<DateTimeFormat(argname, format)>]`.

```fsharp
type EndPoint =
  | [<DateTimeFormat("thedate", "yyyy-MM-dd")>]
    Article of id: int * thedate: DateTime

(* Accepted Request:    GET /Article/43/2015-04-15

   Parsed endpoint: *)  Article(43, System.DateTime(2015, 4, 15, 0, 0, 0,
                                         System.DateTimeKind.Utc))
```

Setting the format of a DateTime record field with `[<DateTimeFormat(format)>]`.

```fsharp
type EndPoint =
  { id: int
    [<DateTimeFormat("yyyy-MM-dd")>]
    date: DateTime }

(* Accepted Request:    GET /Article/43/2015-04-15

   Parsed endpoint: *)  { id = 43; date = System.DateTime(2015, 4, 15, 0, 0, 0,
                                              System.DateTimeKind.Utc) }
```

## Wildcards

`[<Wildcard>]` on a union case encodes the remainder of the path in the last argument.

If the argument's type is `list<'T>` or `array<'T>`, then the path is parsed as a sequence of `'T`.

```fsharp
type EndPoint =
  | [<Wildcard>] Articles of int * list<string>
  | [<Wildcard>] Articles2 of list<int * string>

(* Accepted Request:    GET /Articles/12/abc/def

   Parsed endpoint: *)  Articles (12, ["abc"; "def"])

(* Accepted Request:    GET /Articles/34

   Parsed endpoint: *)  Articles (34, [])

(* Accepted Request:    GET /Articles2/12/abc/34/def

   Parsed endpoint: *)  Articles2 [| (12, "abc"); (34, "def") |]
```

If the argument's type is `string`, then the remainder of the path is passed.

```fsharp
type EndPoint =
  | [<Wildcard>] Articles of int * string

(* Accepted Request:    GET /Articles/12/abc/def

   Parsed endpoint: *)  Articles (12, "abc/def")

(* Accepted Request:    GET /Articles/34

   Parsed endpoint: *)  Articles (34, "")
```

## HTTP Methods

Selected with `[<EndPoint(methodAndFragment)>]` or with `[<Method(name...)>]`.

```fsharp
type EndPoint =
  | [<EndPoint "GET /api">]
    ApiGet of int
  | [<EndPoint "POST /api">]
    ApiPost of int

(* Accepted Request:    GET /api/12

   Parsed endpoint: *)  ApiGet 12

(* Accepted Request:    POST /api/12

   Parsed endpoint: *)  ApiPost 12

type EndPoint =
  | [<Method "GET"; EndPoint "/api">]
    ApiGet of int
  | [<Method "POST"; EndPoint "/api">]
    ApiPost of int

(* Accepted Request:    GET /api/12

   Parsed endpoint: *)  ApiGet 12

(* Accepted Request:    POST /api/posted

   Parsed endpoint: *)  ApiPost int
```

## GET Query parameters

For union case arguments, indicated with `[<Query(argname...)>]`.

```fsharp
type EndPoint =
  | [<Query("start", "count")>]
    Get of id: int * start: int * count: int option

(* Accepted Request:    GET /Get/14?start=3&count=5

   Parsed endpoint: *)  Get(14, 3, Some 5)

(* Accepted Request:    GET /Get/14?start=3

   Parsed endpoint: *)  Get(14, 3, None)
```

For record fields, indicated with `[<Query>]`.

```fsharp
type EndPoint =
  { [<Query>] x: int; y: string }

(* Accepted Request:    GET /test?x=1

   Parsed endpoint: *)  { x = 1; y = "test" }
```

## JSON body

See the [JSON cheat sheet](Json-Ref.md) for the JSON format.

For union case arguments, indicated with `[<Json(argname)>]`.

```fsharp
type EndPoint =
  | [<Method "POST"; Json "body">]
    Article of id: int * body: Body

and Body =
  { x: int
    y: string }

(* Accepted Request:    POST /Article/31

                        {"x":4,"y":"a"}

   Parsed endpoint: *)  Article(31, { x = 4; y = "a" })
```

For record fields, indicated with `[<Json>]`.

```fsharp
type EndPoint =
  | [<Method "POST">]
    Article of ArticleArgs

and ArticleArgs =
  { id: int
    [<Json>] body: Body }

and Body =
  { x: int
    y: string }

(* Accepted Request:    POST /Article/31

                        {"x":4,"y":"a"}

   Parsed endpoint: *)  Article({ id = 31; body = { x = 4 y = "a" } })
```

## Form body

Parses form data with content type `application/x-www-form-urlencoded` or `multipart/form-data`.

For union case arguments, indicated with `[<FormData(argname)>]`.

```fsharp
type EndPoint =
  | [<Method "POST"; FormData("x", "y")>]
    Article of id: int * x: int * y: string

(* Accepted Request:    POST /Article/31
                        Content-Type: application/x-www-form-urlencoded

                        x=4&y=a

   Parsed endpoint: *)  Article(31, 4, "a")

(* Accepted Request:    POST /Article/31
                        Content-Type: multipart/form-data

                        ------WebKitFormBoundaryDcVW4TSGc9MaMCl0
                        Content-Disposition: form-data; name="x"

                        4
                        ------WebKitFormBoundaryDcVW4TSGc9MaMCl0
                        Content-Disposition: form-data; name="y"

                        a
                        ------WebKitFormBoundaryDcVW4TSGc9MaMCl0--

   Parsed endpoint: *)  Article(31, 4, "a")
```

For record fields, indicated with `[<FormData>]`.

```fsharp
type EndPoint =
  | [<Method "POST">]
    Article of ArticleArgs

and ArticleArgs =
  { [<FormData>] x: int
    [<FormData>] y: string }

(* Accepted Request:    POST /Article
                        Content-Type: multipart/form-data

                        ------WebKitFormBoundaryDcVW4TSGc9MaMCl0
                        Content-Disposition: form-data; name="x"

                        4
                        ------WebKitFormBoundaryDcVW4TSGc9MaMCl0
                        Content-Disposition: form-data; name="y"

                        a
                        ------WebKitFormBoundaryDcVW4TSGc9MaMCl0--

   Parsed endpoint: *)  Article({ x = 4; y = "a" })
```
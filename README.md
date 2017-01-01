# `ocaml-graphql-server`

`ocaml-graphql-server` is a library for creating GraphQL servers. It's currently in a very early, experimental stage.

Current feature set:

- [x] GraphQL parser in pure OCaml using [angstrom](https://github.com/inhabitedtype/angstrom) (April 2016 RFC draft)
- [x] Basic execution
- [x] Basic introspection
- [x] Argument support
- [x] Lwt support
- [x] Async support
- [x] Example with HTTP server and GraphiQL

![GraphiQL Example](https://cloud.githubusercontent.com/assets/2518/20041922/403ed918-a471-11e6-8178-1cd22dbc4658.png)

## Documentation

[API documentation](https://andreas.github.io/ocaml-graphql-server/)

## Examples

### GraphiQL

To run a sample GraphQL server also serving GraphiQL, do the following:

```bash
git checkout git@github.com/andreas/ocaml-graphql-server.git
cd ocaml-graphql-server
opam pin add graphql-server .
cd examples
ocamlbuild -use-ocamlfind server.native && ./server.native
```

Now open [http://localhost:8080](http://localhost:8080).

### Defining a Schema

```ocaml
open Graphql

type role = User | Admin
type user = {
  id   : int;
  name : string;
  role : role;
}

let users = [
  { id = 1; name = "Alice"; role = Admin };
  { id = 2; name = "Bob"; role = User }
]

let role = Schema.enum
  ~name:"role"
  ~values:[(User, "user"); (Admin, "admin")]

let user = Schema.(obj
  ~name:"user"
  ~fields:[
    field "id"
      ~typ:(non_null int)
      ~args:Arg.[]
      ~resolve:(fun () p -> p.id)
    ;
    field "name"
      ~typ:(non_null string)
      ~args:Arg.[]
      ~resolve:(fun () p -> p.name)
    ;
    field "role"
      ~typ:(non_null role)
      ~args:Arg.[]
      ~resolve:(fun () p -> p.role)
  ]
)

let schema = Schema.(schema 
    ~fields:[
      field "users"
        ~typ:(non_null (list (non_null user)))
        ~args:Arg.[]
        ~resolve:(fun () () -> users)
    ]
)
```

### Running a Query

```ocaml
let query = Graphql_parser.parse some_string in
Graphql.Schema.execute schema ctx query
```

### Lwt Support

```ocaml
open Lwt.Infix
open Graphql_lwt

let schema = Schema.(schema
  ~fields:[
    field "wait"
    ~typ:(non_null float)
    ~args:Arg.[
      arg "duration" ~typ:float;
    ]
    ~resolve:(fun () () -> Lwt_unix.sleep duration >|= fun () -> duration)
  ]
)
```

### Async Support

```ocaml
open Core.Std
open Async.Std
open Graphql_async

let schema = Schema.(schema
  ~fields:[
    field "wait"
    ~typ:(non_null float)
    ~args:Arg.[
      arg "duration" ~typ:float;
    ]
    ~resolve:(fun () () -> after (Time.Span.of_float duration) >>| fun () -> duration)
  ]
)
```

## Design

Only valid schemas should pass the type checker. If a schema compiles, the following holds:

1. The type of a field agrees with the return type of the resolve function.
2. The arguments of a field agrees with the accepted arguments of the resolve function.
3. The source of a field agrees with the type of the object to which it belongs.
4. The context argument for all resolver functions in a schema agree.

The following types ensure this:

```ocaml
type ('ctx, 'src) obj = {
  name   : string;
  fields : ('ctx, 'src) field list;
}
and ('ctx, 'src) field =
  Field : {
    name    : string;
    typ     : ('ctx, 'out) typ;
    args    : ('out, 'args) Arg.arg_list;
    resolve : 'ctx -> 'src -> 'args;
  } -> ('ctx, 'src) field
and ('ctx, 'src) typ =
  | Object      : ('ctx, 'src) obj -> ('ctx, 'src option) typ
  | List        : ('ctx, 'src) typ -> ('ctx, 'src list option) typ
  | NonNullable : ('ctx, 'src option) typ -> ('ctx, 'src) typ
  | Scalar      : 'src scalar -> ('ctx, 'src option) typ
  | Enum        : 'src enum -> ('ctx, 'src option) typ
```

The type parameters can be interpreted as follows:

- `'ctx` is a value that is passed all resolvers when executing a query against a schema,
- `'src` is the domain-specific source value, e.g. a user record,
- `'args`
- `'out` is the result of the resolver, which must agree with the type of the field.

Particularly noteworthy is `('ctx, 'src) field`, which hides the type `'out`. The type `'out` is used to ensure that the output of a resolver function agrees with the input type of the field's type.

For introspection, three additional types are used to hide the type parameters `'ctx` and `src`:

```ocaml
  type any_typ =
    | AnyTyp : (_, _) typ -> any_typ
    | AnyArgTyp : (_, _) Arg.arg_typ -> any_typ
  type any_field =
    | AnyField : (_, _) field -> any_field
    | AnyArgField : (_, _) Arg.arg -> any_field
  type any_arg = AnyArg : (_, _) Arg.arg -> any_ar
```

This is to avoid "type parameter would avoid it's scope"-errors.
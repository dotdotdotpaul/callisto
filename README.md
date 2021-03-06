# Callisto

  An Elixir library for managing graph database records with Cypher.

  While it shares some similarities with Ecto, it's also a different beast.  Graph databases tend to be schemaless, so defining schemas on top of nodes isn't always useful.  In addition, nodes and relationships (Vertex and Edge here) have two kinds of data:  labels/type and properties.  A Vertex can have multiple labels.  While a label/type can be thought of like a table name, they aren't exactly the same.  A Vertex can have multiple labels, and not all labels have to have the same mix of labels.  It's entirely possible for one Vertex to have a Foo label, another to have a Foo AND Bar label, and another to have a Bar label.

  So, we work with Vertex and Edge structures all the time, but you can define modules to use Properties and they can be used as structures and validators.

## Installation

  If [available in Hex](https://hex.pm/docs/publish), the package can be installed as:

  1. Add `callisto` to your list of dependencies in `mix.exs`:

    ```elixir
    def deps do
      [{:callisto, "~> 0.1.0"}]
    end
    ```

  2. Ensure `callisto` is started before your application:

    ```elixir
    def application do
      [applications: [:callisto]]
    end
    ```

## Configuration

  You'll need to add to your `config.exs` (or `dev.exs`, etc) the configuration for your graph database.  Here's an example when using Neo4j:

  ```elixir
  config :my_app, MyApp.Graph,
    adapter: Callisto.Adapters.Neo4j,
    url: "http://localhost:7474",
    pool_size: 5,
    max_overflow: 2,
    timeout: 30
  ```

  Then you'll want to add it to your application and supervisor tree

  ```
  # in mix.exs
  def application do
    [mod: {MyApp, []},
     applications: [:neo4j_sips, :callisto]] # Presumes you're using Neo4j.
  end
  ```

  ```elixir
  # in lib/my_app.ex
  children = [
    ...
    supervisor(MyApp.Graph, []),
  ]
  ```

## Set up your repository

  To avoid confusion with Ecto repositories (and we figure it's quite likely an app will have connections to both a conventional SQL or NoSQL database AND a graph database), the Callisto repository is Callisto.GraphDB.  Set up your local graph DB handle:

  ```elixir
  defmodule MyApp.Graph do
    use Callisto.GraphDB, otp_app: :my_app
  end
  ```

## Query for stuff

  You should be able to pass Cypher queries into your graph now, and get back a list of hashes, if anything was returned:

  ```elixir
  MyApp.Graph.query("MATCH (v:Foo) RETURN v")
  ```

  But that's where Callisto.Query comes in -- it's intended to represent a query as component parts, which can be reworked over time.  It's also ready to automagically convert Vertex and Edge records into match constructs:

  ```elixir
  alias Callisto{Query, Vertex}
  matcher = Vertex.cast("Foo", id: 42)
  query = %Query{}
          |> Query.match(vert: matcher)
          |> Query.returning(vert: Vertex)
  MyApp.Graph.query(query)
  ```

  If you `to_string` the query, you can see the cypher that results:

  ```MATCH (vert:Foo {id: 42}) RETURN vert```

## Properties

  If you need to validate that your vertices (or edges) have specific properties, or if you want to default properties on them when they are instanced, you can define a properties module, then pass this as a label (or type for edge) and the system will automagically use it to check for fields.

  ```elixir
  defmodule MyApp.FooVertex do
    use Callisto.Properties

    def gen_search_key(data), do: to_string(data.biff) |> String.downcase

    properties id: false, do
      name: "Foo", # Defaults to last term of module, FooVertex
      field: :bar, :integer, default: 42
      field: :biff, :string, required: true
      field: :search_key, :string, default: &gen_search_key/1
    end
  end

  foo = Vertex.new(MyApp.FooVertex, biff: "boom")
  Cypher.to_cypher(foo) # "(x:Foo {bar: 42, biff: 'boom'})"
  ```

  Required means the key HAS to be provided when calling new(); default will only apply if the key does not exist AFTER validation.  It can take a function argument; this is expected to be a 1-arity function, which will be passed the properties hash.  In the above example, the search_key will be defaulted to the downcased version of of the :biff field.

  If you want to instance a Vertex with a specific Properties struct, but don't want to apply the defaults or validations yet (for example, because you're pulling data from the database that may not be valid according to the latest implementation), you can use `cast` instead of `new`.

  Fields defined in a properties block will be converted to atom keys in the props Map stored on the Vertex/Edge struct -- string keys in the Map that are not fields will be left as strings, so as not to pollute the atom space.

## Work in Progress

  This API is evolving as we are starting to use it for an application we're building.  No guarantee we won't completely redo some of the concepts tomorrow.

  Things coming soon:

  * Actually write things to the database with nice CRUD functions (theoretically, you can already write things by crafting your own queries, but we figure it'd be good to be able to do something like "Vertex.new(Foo) |> Graph.save()"
  * Better 'changeset' handling
  * Let us know on GitHub if you have any other ideas!

## Running tests

  To run tests use `mix test` (to run once and exit) or `mix test.watch` (to
  run and wait for changes, then repeat, see https://hex.pm/packages/mix_test_watch 

  To run tests that generate coverage reports, use `mix test --cover`.  By default HTML-based reports are written to the the `cover` folder, and text-based summaries are written to console.

  

## Copyright and License

  Copyright (c) 2016, CollabRx, Inc.

  Callisto source code is licensed under [Apache 2 License](LICENSE.md).

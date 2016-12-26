Change log
==========

## 0.6.0 – Unreleased

TL;DR: Many big changes in how context types work and how they
interact with the executor. Not too much to worry about if you're only
using the macros and not deriving `GraphQLType` directly.

### Breaking changes

* The `executor` argument in all resolver methods is now
  immutable. The executor instead uses interior mutability to store
  errors in a thread-safe manner.

  This change could open up for asynchronous or multi-threaded
  execution: you can today use something like rayon in your resolve
  methods to let child nodes be concurrently resolved.

  **How to fix:** All field resolvers that looked like `field
   name(&mut executor` now should say `field name(&executor`.

* The context type of `GraphQLType` is moved to an associated type;
  meaning it's no longer generic. This only affects people who
  implement the trait manually, _not_ macro users.

  This greatly simplifies a lot of code by ensuring that there only
  can be one `GraphQLType` implementation for any given Rust
  type. However, it has the downside that support for generic contexts
  previously used in scalars, has been removed. Instead, use the new
  context conversion features to accomplish the same task.

  **How to fix:** Instead of `impl GraphQLType<MyContext> for ...`,
   you use `impl GraphQLType for ... { type Context = MyContext;`.

* All context types must derive the `Context` marker trait. This is
  part of an overarching change to allow different types to use
  different contexts.

  **How to fix:** If you have written e.g. `graphql_object!(MyType:
   MyContext ...)` you will need to add `impl Context for MyContext
   {}`. Simple as that.


### Added

* Support for different contexts for different types. As GraphQL
  schemas tend to get large, narrowing down the context type to
  exactly what a given type needs is great for
  encapsulation. Similarly, letting different subsystems use different
  resources thorugh the context is also useful for the same reasons.

  Juniper supports two different methods of doing this, depending on
  your needs: if you have two contexts where one can be converted into
  the other _without any extra knowledge_, you can implement the new
  `FromContext` trait. This is useful if you have multiple crates or
  modules that all belong to the same GraphQL schema:

  ```rust

  struct TopContext {
    db: DatabaseConnection,
    session: WebSession,
    current_user: User,
  }

  struct ModuleOneContext {
    db: DatabaseConnection, // This module only requires a database connection
  }

  impl Context for TopContext {}
  impl Context for ModuleOneContext {}

  impl FromContext<TopContext> for ModuleOneContext {
    fn from(ctx: &TopContext) -> ModuleOneContext {
      ModuleOneContext {
        db: ctx.db.clone()
      }
    }
  }

  graphql_object!(Query: TopContext |&self| {
    field item(&executor) -> Item {
      executor.context().db.get_item()
    }
  });

  // The `Item` type uses another context type - conversion is automatic
  graphql_object!(Item: ModuleOneContext |&self| {
    // ...
  });

  ```

  The other way is to manually perform the conversion in a field
  resolver. This method is preferred when the child context needs
  extra knowledge than what exists in the parent context:

  ```rust

  // Each entity has its own context
  struct TopContext {
    entities: HashMap<i64, EntityContext>,
    db: DatabaseConnection,
  }

  struct EntityContext {
    // fields
  }

  impl Context for TopContext {}
  impl Context for EntityContext {}

  graphql_object!(Query: TopContext |&self| {
    // By returning a tuple (&Context, GraphQLType), you can tell the executor
    // to switch out the context for the returned value. You can wrap the
    // tuple in Option<>, FieldResult<>, FieldResult<Option<>>, or just return
    // the tuple without wrapping it.
    field entity(&executor, key: i64) -> Option<(&EntityContext, Entity)> {
      executor.context().entities.get(&key)
        .map(|ctx| (ctx, executor.context().db.get_entity(key)))
    }
  });

  graphql_object!(Entity: EntityContext |&self| {
    // ...
  });

  ```

## [0.5.3] – 2016-12-05

### Added

* `jtry!`: Helper macro to produce `FieldResult`s from regular
  `Result`s. Wherever you would be using `try!` in a regular function
  or method, you can use `jtry!` in a field resolver:

  ```rust
  graphql_object(MyType: Database |&self| {
    field count(&executor) -> FieldResult<i64> {
      let txn = jtry!(executor.context().transaction());

      let count = jtry!(txn.execute("SELECT COUNT(*) FROM user"));

      Ok(count[0][0])
    }
  });
  ```

### Changes

* Relax context type trait requirements for the iron handler: your
  contexts no longer have to be `Send + Sync`.

* `RootNode` is now `Send` and `Sync` if both the mutation and query
  types implement `Send` and `Sync`.

### Bugfixes

* `return` statements inside field resolvers no longer cause syntax
  errors.



## 0.5.2 – 2016-11-13

### Added

* Support for marking fields and enum values deprecated.
* `input_object!` helper macro

### Changes

* The included example server now uses the simple Star Wars schema
  used in query/introspection tests.

### Bugfixes

* The query validators - particularly ones concerned with validation
  of input data and variables - have been improved significantly. A
  large number of test cases have been added.

* Macro syntax stability has also been improved. All syntactical edge
  cases of the macros have gotten tests to verify their correctness.


[0.5.3]: https://github.com/mhallin/juniper/compare/0.5.2...0.5.3
## FutureExpression

    class FutureExpression extends Array  # shouldn't extend Array?

      { NORMAL, BLOCK, DYNAMIC, MUTABLE, INFINITE } =
          assign this, FUTURE_EXPRESSION_ATTRIBUTES


### Supporting classes

      CompositionConstructor: null
      InvocationConstructor: null



### Constructor

      constructor: ( @attributes = NORMAL, env, name, params, body ) ->
        unless not env? or env instanceof EnvironmentRecord
          throw new TypeError "Invalid parent environment"
        @parentEnvironment = env or = null

        unless typeof name is 'string'
          name = if name? then "#{name}" else ''
        @name = name

        if typeof params is 'string'
          params = params.replace( /^\s*(.*?)\s*$/, '$1' ).split( /\s*,\s*/ )
        @params = params

        if typeof body is 'function' then unless attributes & DYNAMIC
          body = bind body, this
        @body = body



### Class-private


#### bind

Performs the one-time static application of a provided `bodyFunction` in the
context of the `futureExpression` to which it belongs.

(Compare this use of closure to implementations of “classes” in CoffeeScript,
TypeScript, etc.)

Returns a function that, when provided with a lexical `environment` record,
will return an array of functions and nested future expressions, based on this
future expression’s array of functions. Each nested future expression is
created with `environment` as its parent environment.

      bind = ( bodyFunction, futureExpression ) ->
        staticBody = bodyFunction.apply futureExpression
        ( environment ) ->
          for i, fn in body = staticBody[..]
            if fn.returns?.prototype instanceof FutureExpression
              body[i] = fn.apply environment
          body



### Methods


#### do

Pushes a function `fn` directly onto the expression’s body.

      do: ( fn ) ->
        throw TypeError unless typeof fn is 'function'
        @push fn
        fn


#### catch

Boxes `fn` as a contingency element, along with an optional conditional
`predicate`, and then `push`es the element object onto the body.

During an invocation of `this` expression, any exceptions thrown by a
synchronous operation, or any rejections issued by an asynchronous operation,
will cascade down the invocation to the nearest suitable `catch` element.

      catch: ( predicate, fn ) ->
        throw TypeError unless typeof fn is 'function'
        element = { type: 'catch', predicate, value: fn }
        @push element
        element


#### invoke

Invokes `this` `FutureExpression`, returning a new `FutureInvocation`.

Retains the `caller` from which `this` is being invoked, so as to enable
construction of an invocation graph.

* `caller`:`FutureInvocation`
* `args`:`Array`

      invoke: ( caller, args ) ->
        @invocation = new @Invocation caller, this, args


#### apply

      apply: ->
        @invoke( null, arguments ).promise()


#### call

      call: ( args... ) ->
        @invoke( null, args ).promise()



### Expression factories

This section is populated with placeholder functions, included here only for
continuity and expository purposes; each is procedurally overwritten later in
the compilation process by the `Expression methods` submodule.

The implemented methods return a factory function which, when invoked during a
`FutureInvocation` in the context of an associated `EnvironmentRecord`, will
create a new future expression of the specified type, lexically bound to that
environment.

      pipeline: ( name, params, body ) ->
      concurrency: ( name, params, body ) ->
      multiplex: ( name, width, params, body ) ->
      mutableConcurrency: ->
      mutableMultiplex: ->
      blockPipeline: ( params, body ) ->
      blockConcurrency: ( params, body ) ->
      blockMultiplex: ( width, params, body ) ->
      mutableBlockConcurrency: ->
      mutableBlockMultiplex: ->



### Distribution (?) factories

Use pipelines to iterate. Use concurrencies to distribute.

      next: ->

      each: ( collection, distributor, params, block ) ->

* For each element of `collection`
* in the manner of `distributor`, a reference to one of the future
  factory methods of this prototype (e.g. `blockPipeline`,
  `mutableBlockConcurrency`, etc.)
* with enclosed `params`
* invoke the future block returned by `block`

        blocks = ( block for k of collection )
        distributor.call this, params, -> @push blocks

        @blockConcurrency null, ->
          @push block for k of collection


#### each

Returns a function that creates and asynchronously invokes an `iterant` block
`FutureExpression` of the type returned by the named `iterateMethod`, over each
element of a collection, in the manner of the type of block expression returned
by the named `distributeMethod`.

  * `collector` – Function to be invoked in context of the local environment,
    returning an array or object as the collection whose elements will each be
    the subject of an invocation of the iterant block expression.

  * `iteratorMethod` – The method of this prototype to be used to create the
    `iterator`, a block expression that will distribute the invocations of
    the `iteratee` block over the elements of the collection.

  * `iterateeMethod` – The method of this prototype to be used to create the
    `iteratee`, a block expression that will be invoked for each element in the
    collection.

  * `params` – String array (or `null`) containing the formal parameter list
    to be provided to `iterateeMethod`.

  * `body` – Function that will construct the `iteratee` block expression.

      each: ( collector, iteratorMethod, iterateeMethod, params, body ) ->
        # this : FutureExpression

        if typeof iteratorMethod is 'string'
          iteratorMethod = @[ iteratorMethod ]
        if typeof iterateeMethod is 'string'
          iterateeMethod = @[ iterateeMethod ]

Create the function that will return the block future expression to be iterated
once it is applied in the context of an `EnvironmentRecord`.

        # : EnvironmentRecord -> FutureExpression { block: true }
        iterateeConstructor = iterateeMethod.call this, params, body

Create the function that will return the block future expression that will
distribute invocations of the block to be iterated over the elements of the
collection.

        # : EnvironmentRecord -> FutureExpression { block }
        iteratorConstructor = iteratorMethod.call this, null, ->
          # this : EnvironmentRecord
          index = -1
          collection = collector.apply this, arguments
          @next = if isArray collection
          then -> collection[ index += 1 ]
          else do ->
            keys = keysOf collection
            -> collection[ keys[ index += 1 ] ]

The returned function, during a `FutureInvocation` of `this` expression, will
be applied in the context of that invocation’s `EnvironmentRecord`.

each :: ... -> ( (EnvironmentRecord) -> [input] -> Promise output )

        ->
          # this : EnvironmentRecord
          iteratee = iterateeConstructor.call this
          iterator = iteratorConstructor.call this

Invoke `iteratee` over the elements of `collection`, in the manner prescribed
by the distributor.

          iteratee.invoke v, k, o for k, v of o

Invoke the block expression in the manner befitting `distributor`, once per
iterable element of `collection`, with arguments `value`, `key`, and
`collection`.

          iterator.next iteratee

      each: ( iterator, iterationMethod, iterateeMethod, params, body ) ->
        return unless typeof iterator?.next is 'function'

iterationMethod must yield a dynamic block

        iterationBlockConstructor = iterationMethod.call this, null, ->


        iterateeBlockConstructor = iterateeMethod.call this, params, body



> The problem here is that an expression's body must be representable not just as an array, but also as an Iterator. This hides the implementation detail of the collection. This can also allow for async `next`.

> Here a proper Iterator could simply close over `collection`, accept the block expression `iteratee`, and let the runtime call its `next` ad finitum to acquire the values to pass to each invocation of `iteratee`.




#### while

  * `predicate` — Function to be evaluated in context of the local environment.
    Evaluation is asynchronous if `predicate` returns a future.

  * et al, see `each`

Returns a function that creates and asynchronously invokes an iterant block
`FutureExpression` until its `predicate` evaluates negatively.

      while: ( predicate, distributor, iterator, params, body ) ->

if value resembles Future, `value.then` `block` to it and return
  else

        factory = ->
          if 1 then ;
          d = new Deferral
          predicate.then (->), (->)
          block.apply this, arguments



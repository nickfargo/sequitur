## EnvironmentRecord

Function elements of a future invocation are each invoked in the context of a
unique and common **lexical environment**, which is wholly represented here as
an `EnvironmentRecord`. Variables scoped to the invocation may be bound as
typical key-value properties of this object; likewise, members of the prototype
may be used to analogize language-level keywords, control statements, etc.

Nested future expressions retain a reference to their parent environment, such
that any invocation of a nested future will spawn a new `EnvironmentRecord`
that prototypally inherits from the parent environment, thereby analogizing
closures.

    class EnvironmentRecord

A lexical environment is best implemented as a completely “clean” object, i.e.
without lineage to `Object.prototype`. (Not supported pre-ES5, in which case
we just live with the “dirty” environments.)

      @:: = create null



### Class functions


#### create

Provides an `invocation` with a new `EnvironmentRecord` that inherits from the
environment bound to the `FutureExpression` from which the invocation was
spawned. Any arguments provided by `args` are automatically bound to the
corresponding `params` defined by the `expression`.

      @create = ( invocation, args ) =>
        { expression } = invocation
        { parentEnvironment, params } = expression

        environment = if parentEnvironment
        then create parentEnvironment
        else new this

        environment[ identifier ] = args[i] for identifier, i in params
        environment.__invocation__ = invocation
        environment.__expression__ = expression
        environment



### Control methods


#### return

Jumps out of the current invocation of the prevailing future expression,
skipping past any block expressions.

      return: ( value ) ->
        invocation = @__invocation__
        while invocation.expression instanceof BlockExpression
          invocation.resolve value
          invocation = invocation.caller

      yield: ->

      break: ->

      continue: ->



### Error propagation methods


#### throw

Return control to the nearest containing future invocation in the `Error`
state, instructing it to remain in the `Error` state, and resume with the
element after the `catch` that handled the error, continuing the search for
another suitable `catch` downstream.

      throw: ->


#### raise

Reject the prevailing future invocation immediately, returning control to its
containing future invocation, if any. (This is the same behavior as `throw`,
except `raise` won’t look for downstream `catch`es in the local invocation.)

      raise: ->



### Error recovery methods

From within a `catch` block, return control to the nearest containing future
invocation in the `Error` state, instructing it to recover to the `Running`
state, and resume with the element that ...


#### drop

... comes after the `catch` that handled the error. (The `catch`er “drops”
control down to its lexical successor.)

      drop: ->


#### resume

... comes after the element that caused the error.

      resume: ->


#### retry

... caused the error.

      retry: ->



### Iteration methods


#### each

      each: ->


#### invoke

Invokes a future `expression` with the provided `args`, specifying `this`
environment’s bound invocation as the caller.

* `expression` : `FutureExpression`
* `args` : `Array`

      invoke: ( expression, args ) -> expression.apply @__invocation__, args

      invoke_from_tail_position: ( expression, args ) ->  # ????



### Method sets and tagging

Define the set of “control statements” by copying the named prototype methods
to a separate static field. This object is used by a `FutureInvocation` when
one of its function elements is applied and returns a direct reference to one
of these methods in order to effect the corresponding jump behavior.

      @control = do ( out = {} ) =>
        keys = 'return yield break continue throw raise resume retry'
        for k in keys.split /\s+/
          m = out[k] = @::[k]
          m.type = 'control'
          m.token = k
        out



### Expression methods

This section is populated with placeholder functions, included here only for
continuity and expository purposes; each is procedurally overwritten later in
the compilation process by the `Expression methods` submodule.

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

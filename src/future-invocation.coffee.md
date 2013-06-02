## FutureInvocation

An **invocation** is an active calling session of a `FutureExpression`, in
which the function elements that comprise the `expression` body are applied in
the context of an `environment` record that inherits from the environment of
`expression`’s parent expression. Invocations wrap a `future` that is resolved
once the invocation has completed.

> This should be an Iterator implementation


    class FutureInvocation extends Future

      createEnvironmentFor = EnvironmentRecord.create


### Constructor

* `caller`:`FutureInvocation` — The prevailing active invocation.
* `expression`:`FutureExpression` — The expression being invoked.
* `args`:`Array` — The provided arguments.

      constructor: ( @caller, @expression, args ) ->
        { CompositionConstructor } = expression

        @environment = environment = createEnvironmentFor this, args

This invocation’s `body` is generated from the expression’s static body. Before
being passed to the internal future `composition`, a reference to the `body` is
held here, in case it needs to be mutated later, e.g. to accommodate proper
tail calls.

        @body = body = expression.body environment

The internal `FutureComposition` to be resolved later according to the
proceedings of the executable contents of `body`.

        @composition = composition = new CompositionConstructor body
        composition.invocation = this

The execution pointer to the currently active element of `body`.

        @index = 0



### Class-private


#### trampolines

The `trampolines` are a set of functions that each facilitate a particular type
of jump to elsewhere in the invocation graph.

      trampolines = do ->

        recoveryMethod = ( methodName ) -> ->
          catcher = @innermostCatchInvocation()
          unless @acceptCallersUpTo catcher, arguments
            throw SyntaxError "Illegal '#{ methodName }' outside 'catch'"
          catcher.caller.state().do methodName

##### return

Any prevailing block invocations are accepted, along with the prevailing
non-block future invocation. Control is returned to that invocation’s caller,
which will proceed normally in its `Running` state.

        return: ->
          @acceptCallersUpTo @innermostProcInvocation(), arguments


TODO: break and continue should explicitly detect loop blocks

##### break

Concludes `this` block invocation and the containing ...?

        break: ->
          throw SyntaxError unless @expression.attributes & BLOCK
          do @accept
          do @caller.accept

##### continue

Concludes only `this` block invocation, allowing others under the containing
invocation to continue.

        continue: ->
          throw SyntaxError unless @expression.attributes & BLOCK
          do @accept

##### pass

Continues `this` invocation by sending the argument values as received on to
the next function.

TODO: should become means for synchronously returning multiple values

        pass: ->
          return

##### throw

Cancels any prevailing block invocations, then returns control to the
prevailing non-block future invocation, which will proceed in its `Error`
state, in search of its next substituent `catch` block.

        throw: ( err ) ->
          inv = @innermostProcInvocation()
          @cancelInvocationsUntil inv, arguments unless this is inv
          inv.future.do 'throw'

##### raise

Rejects any prevailing block invocations, along with the prevailing non-block
future invocation. This results in control being returned to that invocation’s
caller, which will proceed in its `Error` state.

        raise: ( err ) ->
          @rejectCallersUpTo @innermostProcInvocation(), arguments

##### drop

A parent invocation in the `Error` state will `drop` control onto the successor
of the prevailing `catch`.

        drop: recoveryMethod 'drop'

##### resume

For a parent invocation in the `Error` state, control will `resume` at the
successor of the element that caused the error.

        resume: recoveryMethod 'resume'

##### retry

A parent invocation in the `Error` state will `retry` the element that caused
the error.

        retry: recoveryMethod 'retry'



### Methods


#### terminate

Accepts, rejects, or cancels `this` invocation and all caller invocations up
to, but excluding, an ancestral `target`.

      terminate: ->

Clear references and allow `this` to be GC’d

        @environment.__invocation__ = null
        @environment = null
        @body = null
        @future.invocation = null
        @future = null
        this


#### innermostCatchInvocation

      innermostCatchInvocation: ->
        invocation = this
        while invocation = invocation.caller
          return invocation if invocation.expression.catch


#### innermostProcInvocation

      innermostProcInvocation: ->
        invocation = this
        while invocation.expression.attributes & BLOCK
          invocation = invocation.caller
        invocation


#### outermostBlockInvocation

      outermostBlockInvocation: ->
        return null unless @expression.block
        invocation = this
        while ( caller = invocation.caller ).expression.attributes & BLOCK
          invocation = caller
        invocation



### Resolution propagation methods

      make = ( name ) -> ( target, args ) ->

Validate `target` as a parent invocation of `this`.

        return unless target
        invocation = this; loop
          break if invocation is target
          return unless invocation = invocation.caller

        invocation = this; loop
          return if invocation is target
          invocation[ name ].apply invocation, args
          invocation = invocation.caller

      acceptInvocationsUntil: make 'accept'
      rejectInvocationsUntil: make 'reject'
      cancelInvocationsUntil: make 'cancel'


#### evaluate

Sent from the `@future`.

      evaluate: ( value ) ->

A “control statement” effects

        return StopIteration if value.control and @jump value

Functions that return a future expression must be evaluated in the context of
`this` invocation’s `EnvironmentRecord`.

        if value.returns?.prototype instanceof FutureExpression
          value = value.apply @environment

        return value or StopIteration


#### jump

Responds to control statement values encountered by the active `composition` by
applying a matching trampoline function.

      jump: ( value ) ->

The element `value` instigating the `jump` must have a `token` property that
names one of the “control statement” methods.

        return unless ( name = value.token ) of trampolines and
          method = EnvironmentRecord::[ name ]

`value` can be simply a *reference* to a control method, with no accompanying
values, e.g. `@break`; or it can be an object that wraps an array of `values`,
created at runtime by calling the `method`, e.g. `@return 1` or
`@return 1, 2, 3`.

        { values } = value if value isnt method
        trampolines[ name ].call this, values



### Methods that map directly to `@composition`

      then: -> ( composition = @composition ).then.apply composition, arguments

      promise: -> @composition.promise()

      given: ->
        @composition.given arguments
        this

      accept: ->
        ( composition = @composition ).accept.apply composition, arguments
        @end()

      reject: ->
        ( composition = @composition ).reject.apply composition, arguments
        @end()

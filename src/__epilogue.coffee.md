## epilogue

After all is said and done...



### Expression methods

The `EnvironmentRecord` and `FutureExpression` prototypes are augmented with
methods for creating new `FutureExpression`s that are automatically bound to a
prevailing parent environment.

`EnvironmentRecord`’s methods immediately return a new `FutureExpression` where
it is bound as the expression’s parent environment. `FutureExpression`’s
methods return thunks, to be invoked later during a `FutureInvocation` of an
outer expression, which will produce the new inner expression, bound to that
invocation’s associated `EnvironmentRecord`. (Partially applying the method’s
arguments into the thunk makes it convenient to define a `FutureExpression`
from within the `body` of its containing `FutureExpression`.)

    do ->

      constructors = {
        PipelineExpression
        MultiplexExpression
        ConcurrencyExpression
        BlockPipelineExpression
        BlockMultiplexExpression
        BlockConcurrencyExpression
        DynamicPipelineExpression
        DynamicMultiplexExpression
        DynamicConcurrencyExpression
        DynamicBlockPipelineExpression
        DynamicBlockMultiplexExpression
        DynamicBlockConcurrencyExpression
      }

> e.g., `keyFrom 'FooBarExpression'` returns `fooBar`

      keyFrom = do ->
        replacer = ( match, $1, $2 ) -> ( $1 or '' ).toLowerCase() + $2
        (s) -> s.replace /^(\w?)(.*?)Expression$/, replacer

Procedurally define the methods associated with each `FutureExpression`
constructor.

      for name, ExpressionConstructor of constructors when key = keyFrom name
        do ( ExpressionConstructor ) ->

          EnvironmentRecord::[ key ] = ->
            new ExpressionConstructor this, arguments...

          FutureExpression::[ key ] = ( args... ) ->
            thunk = -> new ExpressionConstructor this, args...
            thunk.returns = ExpressionConstructor
            thunk

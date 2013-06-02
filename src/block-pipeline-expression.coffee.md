## BlockPipelineExpression

    class BlockPipelineExpression extends PipelineExpression
      block: true

      constructor: ( parentEnvironment, params, body ) ->
        super parentEnvironment, null, params, body

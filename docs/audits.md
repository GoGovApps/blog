# How we acheived context-enriched audit logs in microservices

@GOGov, we are using Sinatra microservices to compartmentalize and replace our legacy PHP monolith.  We have a sprawling legacy application with much out-dated technology.  
However, we needed to start capturing database level changes for auditing.  Doing so in our legacy application would have been a major challenge, as any development in that environment
introduces a high-risk deployment that is difficult to verify.

Our goal was to create a pattern that would allow us to audit records, and then enrich them with context as we continued to build out microservices.

We implemented [Maxwell](https://github.com/zendesk/maxwell) to provide our infrastructure with three main capabilities:
- Process audit logs
- Process database changes to Elasticsearch
- Potentially to process into a cache invalidation scheme (TBD)

The major issue with using Change Data Capture (CDC) for audit logs is the difficulty in coordinating the database record change with the _user_ that made the change.
Our solution follows.

### Introducing a middleware

We used maxwell to pump our database changes through Kinesis.  We then experimented with various ways of processing that stream with context data.  
We finally landed on a solution, with a [slight improvement](https://github.com/zendesk/maxwell/pull/1533) to Maxwell.  The key was to allow Maxwell to produce messages
to Kinesis by thread_id.  This means we could link changes back to the context they occurred in.  But how did we get the context data through to the Kinesis processor?

Using the concept of a [black hole table](https://dev.mysql.com/doc/refman/8.0/en/blackhole-storage-engine.html), we built a middleware in our Sinatra applications capable of writing an insert statement after any POST, PUT, PATCH request.
Then, downstream, the KCL processor could build a cache of contexts.  When the KCL processor came across a record change it cared about, it could look at the cache 
of contexts and write an audit log with the enriched information.  

The middelware would perform the insert both before and after the actual server request was handled, thus wrapping the database transaction in context data.  Since the PR 
above published data based on Thread ID, changes within the same Thread ID are guaranteed to be processed by the same KCL processor, therefore allowing scalability not
to be prohibted.

```
module Go
  class Cdc
    def initialize app
      @app = app
    end

    def generate_marker(env)
      request = Rack::Request.new(env)
      actor_id = request.env[:token]&.dig('data','user','id')
      context = {
        actor_id: actor_id,
        ip_address: request.forwarded_for || request.ip
      }
      ActiveRecord::Base.connection.execute(
        <<SQL
INSERT INTO cdc_thread_requests (thread_id, context) VALUES (CONNECTION_ID(), '#{JSON.generate(context)}')
SQL
      )
    end

    def should_generate_marker?(env)
      return false if env['REQUEST_METHOD'].downcase.to_sym == :options
      return false if env['REQUEST_METHOD'].downcase.to_sym == :get
      return false if ENV['RACK_ENV'] == 'test'
      true
    end

    def call env
      generate_marker(env) if should_generate_marker?(env)
      response = @app.call(env)
      generate_marker(env) if should_generate_marker?(env)
      response
    end
  end
end
```

To reduce database traffic, we assume our servers are not mutating data on GETs and OPTION requests, which is just basic good practice :)

## Gotchas

### The thread context

Our KCL processor builds a `thread_context` variable in-memory.  This means it is not deploy safe, since a new deployment would ruin the context information.  If a 
transaction spanned a deployment, this would mean it would not have the context for _who_ made the change. 

A simple solution would be to leverage a Redis cache.  This would mean that the context would persist during a rollout.



## Conclusion

Since we are still building out microservices, we have to accept that while we can still add audits to any record, only some of them will be enriched with context information.
However, adding audit logs is now decoupled from our core business logic, and can be easily added to our KCL processor.  

We are excited about this change, and how the implementation requires little application lift. 

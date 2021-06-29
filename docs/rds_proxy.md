Last year, we got up in running with Istio in our cluster.  One issue we ran into was an intermittent "Lost connection during query" error.  The queries were not long nor complex, nor returning vast amounts of data, and there was no substantial timeout involved, the queries failed right away.

Unable to resolve the issue, our team removed sidecars, which seemed to resolve the issue.  However, this obviously hampered our ability to implement Istio.  

During the debugging of this problem, we were not able to find reasonable ways to reproduce the issue.  Our stack consists of Puma, Sinatra, ActiveRecord, MySQL, running in Kubernetes. The mysql2 repo has a number of related issues - but they are all red herrings for this situation.  Additionally, the same held true for stack overflow.

We messed around with idle_timeout, connection_timeout, read_timeout, wait_timeout, etc, but nothing proved to be consequential.  

Eventually, we gave up, because without the sidecars, the issue went away.

More recently, we ran into "Too Many Connections" issues across our cluster in our staging environment, where our database is smaller and we run extensive automated tests.  Hoping to resolve this issue, we introduced RDS Proxy to help manage connections.  However, this introduced "Lost connection during query" back to our cluster.  Oh no!

Previously, we _wanted_ to use Istio, but now we _needed_ RDS Proxy (or did we?).  Again, we scoured the internet for  literally 12 hours in one day.  Again, we tweaked Puma, ActiveRecord, Proxy timeouts to try to reproduce/resolve.  we tried on_worker_booted and after_fork hooks in Puma.  

We finally started making some progress after reducing the RDS Proxy Idle Connection Timeout to 1 minute.  We were able to reproduce locally (FINALLY) which would speed up our debugging process.  After 1 minute, a single threaded, single worker configuration would fail, reconnect, then work.  However if we made no calls for a minute, we could reproduce.  

A theory emerged - were we not closing connections after network requests?  It occurred to our team that maybe we could observe the ActiveRecord pool to see what was going on.  We launched a thread to print `ActiveRecord::Base.connection_pool.stat` and sure enough, after one network request, the thread was not releasing the connection!

This finally led us to the solution, confirmed and eloquently documented by Ivan Sebastian - https://ivantedja.medium.com/investigating-sinatra-active-record-cloud-sql-connection-bottleneck-c97ff2bde02f.  We had not configured our Sinatra microservices correctly.

With the microservices corrected, we were no longer running into the connectivity issues.  And, we were no longer reserving 2(workers)*5(threads)*(X)pods connections per microservice.   Did we need RDS Proxy after all????

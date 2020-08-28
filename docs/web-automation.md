# cucumber-lambda

### Once upon a time...

In my previous role at my last gig, I spear headed an effort to improve our QA process, tying it into the development lifecycle.  At the time, our leadership was using TestRail, a dated application for documenting Gherkin style tests.  All of this documentation, and running of the tests, was manual.  Namely, a QA engineer would run through a test by clicking through the browser and mark it's result in TestRail.

We wanted to have better TestRail documentation, and have the TestRail tests run more frequently, closer to the release of our product Stories.  This is a tall ask: a growing number of manual tests to be executed inline to our development cycle.  Would we really be able to run through hundreds of regression tests without slowing us down?  Unlikely.

I started experimenting with the idea that we could leverage Cucumber and Capybara.  We needed a solid foundation for reliable tests that could be easily writted, were composable for any engineer, digestable for any product manager, and repeatable on any machine.  I wrote the first 5-10 tests, to get a feel for how they could look, and we then started getting the team I was managing on board.  The team was amenable to the idea - regression proof development and increased confidence in deployments. 

As we scaled up the number of tests, the framework for running the tests had to mature.  We matured to `cucumber-parallel`, then to multiple Jenkins runners running `cucumber-parallel`, then to multiple Jenkins nodes running `cucumber-parallel`.  But we needed more as we were breaking through 100 and even 200 tests.  

At the time, AWS Lambdas were not available in Ruby, but all of our tests were written in ruby.  The desire to use Lambdas was very strong, it was obvious that if we could hit all of our tests in parallel, we could run the whole suite in 6 minutes.  This timing is vital, as this automation suite's goal was to be a gating factor in our production deploys.  We overcame a number of engineering challenges to get our cucumber tests to run on JRuby in a Java Lambda.  We also built a runner to invoke the Lambdas and collect the results.  

Soon, AWS released Ruby Lambdas, and we were able to streamline the building and deployment of our Lambdas.  Everyone rejoice!

### Back to GOGov

There's no doubt in my mind that this is a valuable tool for an engineering organization, if managed properly.  In a microservice world, it can provide feedback on regressions that would otherwise fall through the cracks.  It can tell us if changes to infrastructure has a positive or negative impact on performance.  It can pump data through a system that exceeds the future capacity requirements.  

At GOGov, we have been working on a similar suite.  At 0-100 tests, this can be run in serial, taking about 30 minutes once we got to 100.  However, at our scale, this was still feasible, but we needed a plan.

Using the takeaways from the previous Lambda experience, we have built a similar architecture with some serious improvements.  

### The Lambda

In the previous role, any branch of the cucumber suite would require building and deploying a Lambda with the cucumber code baked in.  It also had to be built with Chrome binaries.  This could take upwards of 10-15 minutes to build and deploy.  Code changes would require a rebuild and redeploy of the lambda.  

At GOGov, we made the Lambda agnostic to the cucumber test suite.  In a simple change, the Lambda is actually responsible for downloading the github repository, then running the cucumber test it is told to run.  The Lambda will still perform a `bundle install`, then kick off the cucumber binary.

Of course, there is a problem with this: What about gems with native extensions?  We felt that the gain of an agnostic lambda was too great, and we'd be better off by having the Lambda baked with many common gems.  For example, when the lambda is built, we run `gem install nokogiri --version a.b.c` and `gem install nokogiri --version a.b.d`.  We feel this is a managable tradeoff, given the performance improvements and reduction in complexity.

We also introduced more up to date versions of Chrome by using a [Lambda Layer with chrome baked in](https://github.com/shelfio/chrome-aws-lambda-layer).  You can find the code [here](https://github.com/GoGovApps/web-automation-lambda)

### The Runner

The runner must invoke the lambda with the proper environment variables, feature directive, and information for where to put results.  It is then responsible for waiting for lambdas to complete, and stitching the results back together.  

At GOGov, we designed the runner to look just like the cucumber CLI.  Namely, when a test completes, it dumps the Gherkin in the console, colored red or green indicative of failure/success.  

The runner also manages retries, reducing perceived 'flaky' tests and their impact on the deploy pipeline.  

When the tests complete, the runner will let you know how much time you've saved.  We are at 107 scenarios, and with a concurrency of 25, we are saving about 45 minutes per run!

![Image of cucumber-lambda results](https://s3.amazonaws.com/blog.govoutreach.com/Screen+Shot+2020-08-27+at+11.56.47+PM.png)

You can find the code [here](https://github.com/GoGovApps/cucumber-lambda)

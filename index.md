---
layout: default
---

# Articles

## How we reindex Elasticsearch safely

Elasticsearch is very powerful, but when you are starting a new project, its hard to know how bad your mapping is :D 

In our architecture, we can update our mappings with no downtime.

[Read More](/docs/elasticsearch_reindex.md)


## Building context-enriched audit trails

Building audit logs can be tedious.  But ultimately, the source of truth typically is your database.  Processing post-commit database changes with Kinesis provides an easy-to-maintain solution to tracking any change to any record.  Adding application context is a little bit trickier, but at GOGov, we <3 to engineer solutions.

[Read More](/docs/audits.md)

## White Labeling Apps

Tools like Fastlane make deploying and managing your companies app a breeze.  But how do things change when the app is a dynamic package, "white labeled" and managed on a per customer basis?

[Read More](/docs/whitelabel_cd.md)


## Reflections on an AWS Migration for a Legacy System

In this article, we'll dive in to the process, the learnings, and the experience gained from understanding a system foreign to us, constructing a plan to create a zero (or very low) downtime migration path, and executing on that path.

[Read More](/docs/migration.md)


## Overcoming a Legacy Pattern: PHP Sessions

It used to be common practice to use PHP Sessions, and store lots of information in them as a cache.  Here, we'll discuss how to overcome this legacy anti-pattern and move toward a stateless JWT approach that will work for web and mobile.

[Read More](/docs/sessions.md)


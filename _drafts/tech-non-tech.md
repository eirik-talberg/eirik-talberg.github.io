---
layout: post
title: "When technical problems become non-technical"
categories:
  - architecture
---

![Heading](/assets/tech-non-tech/heading.jpg)

The almighty YouTube algorithm recently recommended me a great talk
by [Jonathan "J" Tower](https://www.linkedin.com/in/jtower/)
called [Don't build a distributed monolith](https://www.youtube.com/watch?v=p2GlRToY5HI), which I found to be a great
watch and highly recommended both for existing and future software architects. I this talk, J succinctly describes the
major properties of a distributed monolith and how you probably accidentally made one. I know I have. Pay special
attention to the diagram at around the **22-minute mark**, which I think visualises the different properties of system
design directions in a very good way.

I think an important take-away from this talk is in line with one of my major philosophical guidelines within software
architecture, system design and thinking about solutions: **Every choice you make has a cost**. Now, that may seem
obvious to you as you read this, but I've often found in my time as a software developer (which I do still think of
myself as) that people tend to gloss over the actual costs of choices they make, especially when they are
motivated to do a certain thing. J touches on this in his talk as well.

We all want to work on cool technology, but we as humans often miss the forest for the trees and get railroaded by
our own motivations. I've seen less than optimal choices being made, on account of it being "cool" or a certain
person on the team reading about it and wanting to play with it. This has come to the detriment of the business at
large, as they are in the following years saddled with the responsibility of maintaining it, which includes finding
talent to work on it. On the other hand, I've seen choices being made solely because "That's the way we've always done
it" and it has crippled a team's ability to innovate and move forward. It can be a hard balance to strike.

# Story time

Once upon a time, I was working with a client that had built up a fairly large organisation, mostly tech related but
some legal and administrative as well. In an effort to improve the general resilience and robustness of their systems,
they had instituted a new architectural direction, based on microservices, communication through events and event
sourcing, and autonomous teams with a large degree of freedom within their domain. This was, in part, a result of there
being way too many hard couplings between data and services across systems and applications within the organisation,
making it necessary for everyone to walk in lock-step whenever changes and upgrades had to be made. The goal was for
everyone to communicate and share data through a central event bus (in this case Kafka), which would facilitate eventual
consistency across the whole problem domain.

I was hired to work on Team A, responsible for a somewhat large application (A). My task was to sketch out a way
forward, allowing us to modernise and partly re-do A from the inside while maintaining full backwards compatibility with
all consumers (quite a few). Enter Team B, with their application B. B needs data owned by A, and this has, so far,
been done with an SFTP file integration, where a batch job regularly sends data from A to B. You might wrinkle your nose
at this, but this was a result of both legal and technical issues at a time when A and B wasn't a part of the same
organisation. Also, just to make it all a little more complication: B shares A's data through APIs to third parties
within the same problem domain. This has an interesting side effect: B is responsible for the delivery and quality of
data it does not own or control to third parties.

![Integrations](/assets/tech-non-tech/integrations.jpg){: width="500" }

### The law

Here comes our first bit of complication. See, there is a large and quite complex legal framework surrounding the kinds
of data these applications are moving around.

* A is legally required to delete data after 12 months (as an example)
* B is legally required to delete data after 18 months, due to archival and historical reasons which allow for longer
  data retention

This means that even in a Service Oriented Architecture (SOA) driven world, B could not fetch data from A through an
API (SOAP or otherwise), since the retention laws are written to be application specific instead of domain specific.
This is, most likely, a product of an ever evolving legal framework surrounding this problem domain which failed
to take into account technical advancements, organisational restructuring and new paradigms for data sharing and
enabling.

### Team motivation and priorities

Team A is tasked with modernising their stack, making some new applications and killing some ones. They find that the
new architectural patterns fits well with the kinds of processes they are responsible for, and want to remove the SFTP
integration and replace it with an event stream that B can listen to and get their updated data quicker.

However, team B has been saddled with keeping and maintaining both data and services that are not really their concern.
They have had to hire people who know A's domain to keep up with what the changes in data mean for their use, and they
are sick of it. They don't want to know anything about A's data, store or be responsible for deleting it, they just want
an API they can fetch and proxy it through.

### Central issues

Team B's concerns are completely valid. They shouldn't have to spend money and resources on understanding A's domain.
They shouldn't have to be responsible for delivering data they do not control themselves either. Let's say that we put
aside the legal obstacles, making it possible for A to deliver data through APIs in accordance with some data retention
rule, enforced by some authorization rules. How does that affect A's delivery pipeline? What if B has a higher SLA
requirement than A? If service B has a hard runtime dependency to A, A is no longer in charge of its own delivery
cadence. We are back into lock-step then, just on a smaller scale.

Such a pattern would also break with the larger architectural strategy of eventual consistency and sharing data through
events. Does team B have the political capital to veer away from the common strategy of the organisation at large, and
commit another team to be a dependency? Then again, if any piece of data you store introduces a full set of custom
data retention laws that you are required to follow, why would you want to touch it, if avoidable?

These are organisational, political and legal problems, not technical ones. But we're developers?

### Closing thoughts

This issue was not resolved by the time I left my assignment with this client. However, major pushes have been made to
resolve it, both through updating the legal framework surrounding these systems and looking at how teams interact
and cooperate to solve the bigger issues. I think this particular challenge highlights the truth of Conway's Law:

> Any organization that designs a system (defined broadly) will produce a design whose structure is a copy of the
> organization's communication structure.
> -Melvin E. Conway

The problem described here is, in my opinion, the combination of a few different factors:

* Old systems design thinking, from silo-based organisations
* Legal frameworks that are formulated to control implementation instead of intent
* An organisational structure that incorporates parts of many smaller organisations that used to operate separately and
  autonomously, but now are organised together

These factor together make for a very complex problem to solve. Maybe the current team structure doesn't properly
reflect the problems the organisations, in its current form, is trying to solve? A's data is clearly a cross-cutting
concern that affects other applications and partners. Then again, in the service of absolute resiliency, uptime and
flexibility, maybe we need to accept that we need to pick up data from someone else so that we can be effective
as a larger unit. Even if we pay with some overhead with regard to ensuring data is properly deleted at the appropriate
times.

This experience taught me that everyone in the room can be honest and approach the problem with the best intent, and
the conclusion can still end up being wrong. As architects, we have to remember to peek above our computer screens and
look at the people and organisations we are working with. If you approach all problems from a technical standpoint, you
can end up missing approaches (like going to the lawyers for help) and you can end up with more technical debt and
complexity than you actually need. These kinds of issues and problem areas, combined with the need for technical
knowledge, is why I love working in this field.

Until next time,

/Eirik
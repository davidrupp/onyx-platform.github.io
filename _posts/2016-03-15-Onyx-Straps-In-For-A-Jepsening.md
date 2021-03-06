---
layout: post
title:  "Onyx Straps in For a Jepsening"
date:   2016-03-15 00:00:00
categories: jekyll update
author: Lucas Bradstreet
---

## Strapping in for a Jepsening

[Onyx](http://www.onyxplatform.org) is a high performance, distributed, fault tolerant, scalable data processing platform. Onyx programs are described in immutable data structures allowing jobs to cross language and machine boundaries at runtime.

### Testing

Distributed systems are incredibly powerful for dealing with massive amounts of load and providing high availability. Ensuring that your system behaves correctly under stress, however, is a notoriously difficult problem. All of this power is useless if you can't trust your system to handle network partitions, connection loss, killed nodes, consistency anomalies, and other nasty issues.

From the beginning, Onyx has had a variety of unit and integration tests. Over time we have also added numerous property tests to the mix. Our property tests stress our peer coordination code paths and cluster scheduler, and we found numerous bugs that would have been hard to pickup with other testing methods. These techniques have allowed us to add complex features quickly.

While we have users happily [using Onyx in production](https://github.com/onyx-platform/onyx#companies-running-onyx-in-production), it is likely that there are bugs waiting for the right set of scenarios to occur. When they do, reproducing these scenarios can be incredibly time consuming. We would much prefer to find these issues early and to have a way to test every release against grueling conditions that may only occasionally occur in a production environment.

Many forms of distributed tests can be both difficult to formulate and time consuming for developers to build. Luckily, a paper, [Simple Testing Can Prevent Most Critical Failures Yuan et. al.](http://www.eecg.toronto.edu/~yuan/papers/failure_analysis_osdi14.pdf) found that almost all distributed systems failures can be reproduced with 3 or fewer nodes. Howevere we were in need of a better way to test for these forms of faults.

Kyle Kingsbury's [Jepsen](https://github.com/aphyr/jepsen) library and [Call Me Maybe](https://aphyr.com/tags/jepsen) series have been blazing a path to better testing of distributed systems. A Jepsen test is self described by Kingsbury as "a Clojure program which uses the Jepsen library to set up a distributed system, run a bunch of operations against that system, and verify that the history of those operations makes sense". Kyle has been dragging the distributed systems world into a more consistent (and pager friendly) future. Did we mention that he's now available [for Jepsen consulting?](http://jepsen.io/)

### Starting out

As the Onyx team was new to Jepsen, we decided to initially perform a trial test on one of our dependencies. Onyx depends on two external services. The first is ZooKeeper, a distributed CP datastore, which we use for Onyx peer coordination, and the second is BookKeeper, a replicated log server, which we use to build replicated aggregation state machines to provide durability and consistency guarantees for our [State Management / Windowing](http://www.onyxplatform.org/docs/user-guide/latest/aggregation-state-management.html) features.

As ZooKeeper has already received the [Call Me Maybe treatment](https://aphyr.com/posts/291-jepsen-zookeeper), and passed with flying colors, we decided to first test BookKeeper. Testing our dependencies first gives us greater certainty about our system, and allows us to be reasonably sure that any bugs we find are our own fault, or will be fixed upstream.

### Setting up our Jepsen Environment

We initially setup our Jepsen environment in the recommended way, by implementing `jepsen.db/DB`'s `setup!` and `teardown!` procedures. Under our initial setup, Jepsen ran commands on each node via ssh, to install and reset ZooKeeper and BookKeeper to their original states. As this process was taking minutes to perform in our docker-in-docker configuration, we found this to be an impediment to test development time.

We were already using docker-in-docker to run our Jepsen nodes, and by adding an additional layer to Jepsen's docker containers we were able to avoid the Jepsen setup and teardown process completely. Each test would spin up a new set of containers in a pristine state - allowing us to iterate our tests quicker. See our Jepsen docker setup [README](https://github.com/onyx-platform/onyx-jepsen/blob/master/docker/README.md) for more information.

### Testing BookKeeper

Jepsen operates by spinning up `n` [server](https://github.com/aphyr/jepsen/blob/master/jepsen/src/jepsen/os.clj) nodes (in our case 5), and `y` [clients](https://github.com/aphyr/jepsen/blob/master/jepsen/src/jepsen/client.clj) (in our case 5). On each of the server nodes we ran a BookKeeper server and a ZooKeeper server, required for BookKeeper's operation.

Our test was configured to have 5 client threads writing to a BookKeeper ledger configured with an [ensemble size](http://www.onyxplatform.org/docs/cheat-sheet/latest/#peer-config/:onyx.bookkeeper/ledger-ensemble-size) of 3, and a [quorum size](http://www.onyxplatform.org/docs/cheat-sheet/latest/#peer-config/:onyx.bookkeeper/ledger-quorum-size) of 3. This is the default configuration used by Onyx in its state management feature.

The 5 client threads write to this ledger, commanded by a a simple generator that generates incrementing values to be written to the ledger.

```clojure
(gen/phases
 (->> (->> (range)
           (map (fn [x] {:type :invoke, :f :add, :value x}))
           gen/seq)
      (gen/stagger 1/10)
      (gen/delay 1)
      (gen/nemesis
        (gen/seq (cycle
                    [(gen/sleep 30)
                     {:type :info :f :start}
                     (gen/sleep 200)
                     {:type :info :f :stop}])))
                     (gen/time-limit 800)) 
      (read-ledger))
```

This generator also commands the nemesis to [partition random halves](https://github.com/aphyr/jepsen/blob/master/jepsen/src/jepsen/nemesis.clj#L99) of the network, and in an alternate test, partition via the [bridge nemesis](https://github.com/aphyr/jepsen/blob/master/jepsen/src/jepsen/nemesis.clj#L59).

The final phase of the test is to read the ledger back. BookKeeper only allows a ledger to be written to by a single ledger handle, and guarantees that values read from a ledger will be in the order that they were written. This makes it rather easy to test for correctness: we simply read back the ledger, and inspect the history of the writes in our Jepsen [Checker](https://github.com/aphyr/jepsen/blob/master/jepsen/src/jepsen/checker.clj).

The first hitch was we had to account for writes to the ledger that were unacknowledged, but read back by the checker. These are allowable and expected - see the [Two Generals Problem](https://en.wikipedia.org/wiki/Two_Generals%27_Problem) - and can be handled at the application layer if required. Onyx ensures that any events that must be transactional for correctness are written in the same write.

After accounting for this checker discrepancy, we were still not able to get full test runs to complete.  The root cause of this issue was simple to determine. Our BookKeeper servers were committing suicide upon losing quorum. While this is a reasonable response to this issue, it was not our assumption, and it is not documented in BookKeeper's documentation. After creating a [JIRA issue](https://issues.apache.org/jira/browse/BOOKKEEPER-882) for this documentation issue, and daemonising the BookKeeper server, we were able to achieve consistently successful test runs! Sometimes, the nemesis would cause all writes to a ledger to fail, however this is the intended behavior under these conditions, and an additional abstraction over a number of BookKeeper ledgers should be built if required. Kudos to the BookKeeper team for passing these tests with only a documentation issue.

### A simple first Onyx test

Having tested our dependencies, our next move was to start testing Onyx itself.

Onyx operates by building a job composed of a workflow DAG of tasks, their configuration, and a scheduler configuration. Onyx depends on having a durable input stream as the input nodes of the DAG so that unprocessed data can be replayed in the case of input peer failures / rescheduling.

Onyx already supports numerous [plugins](http://github.com/onyx-platform/onyx/tree/master#build-status), however we have not Jepsen tested all of the products that they use under partition conditions. Luckily we have already tested BookKeeper and configured BookKeeper to run on Jepsen, and thus we decided to write [onyx-bookkeeper](https://github.com/onyx-platform/onyx-bookkeeper). As a side note, developing an Onyx plugin used in a Jepsen test quickly found issues with our implementation at development time. One [such issue](https://github.com/onyx-platform/onyx/issues/435) was a cross-cutting problem in some of Onyx's other plugins.

Building an Onyx plugin for a Jepsen tested, durable input and output medium allowed us to build Onyx jobs consisting of BookKeeper data sources and sinks. We wrote a function to build a simple Onyx job to read from 1-5 BookKeeper ledgers, pass through an intermediate task that adds the job number to the message so that we could ensure that the segment has been routed from the correct job, and write the resulting segment to new BookKeeper ledgers.

This test can dynamically build Onyx jobs based on a parameter that defines how many jobs should read from the ledgers. As the number of ledgers needs to be split up over the number of jobs, we tested Onyx scheduling 5 simultaneous jobs, reading from one ledger each, as well as 1 job, reading from all 5 ledgers.

A programatically generated job, reading from one ledger, is shown below. In the below case, 1 job is submitted to the cluster. Hover over the tasks to view the configuration of each task.


<iframe src="{{ '/assets/jepsen_viz/basic.html' | prepend: site.baseurl }}" width="960" height="255" scrolling="no"></iframe>

Test configuration:

```clojure
{:job-params {:batch-size 1}
 :job-type :simple-job
 :nemesis :random-halves ; :bridge-shuffle or :random-halves
 :awake-ms 200
 :stopped-ms 100
 :time-limit 2000
 :n-jobs 5
 ; number of Onyx peers per Jepsen node
 :n-peers 3})
```

We configured 3 Onyx [virtual peers](https://github.com/onyx-platform/onyx/blob/0.8.x/doc/user-guide/concepts.md#virtual-peer) to run per Jepsen node (of which there are 5 nodes). Onyx peers are general purpose execution units that can be assigned to tasks in jobs. For example one peer may be allocated to the `:read-ledger-3` task, to read from a BookKeeper ledger, and pass any read messages onto peers assigned to outgoing tasks in the job's directed acyclic graph. 

Each task requires at least one virtual peer to be running. As the generator may submit 5 jobs with 3 tasks each, and each task requires at least one peer to run, it is possible for jobs will be descheduled by Onyx during nemesis events where nodes are partitioned from the ZooKeeper quorum, and re-allocated after healing or if one of the other jobs completes. This means that our scheduler would also be tested by the nemesis.

We also configured each node with a full ZooKeeper instance and a full BookKeeper instance per node. 

Upon completing all of the jobs, the Jepsen checker reads back from the output ledgers, and determines whether all values written by the clients to the input ledgers were processed and written to the output ledgers, including the correct annotation of the job name.

We quickly hit a number of issues, mostly relating to the peers join process, as well as rebooting themselves after being excised from the cluster.

* [Peer join race condition #453](https://github.com/onyx-platform/onyx/issues/453) Resolved.
* [Peers that crash on component/start will not reboot #437](https://github.com/onyx-platform/onyx/issues/437) Resolved. 
* [Ensure peer restarts after ZooKeeper connection loss/errors #423](https://github.com/onyx-platform/onyx/issues/423) Resolved.

While we had property tests to thoroughly test the peer join process, the above bugs mostly appear in the impure sections of our code. These bugs operate in the real world where peers are not always able to write their coordination log entries, do not always manage to call their side effects, etc.

Jepsen uses excellent scientific procedures for running tests, by outputting dated records including a `result.edn` file, and history to a timestamped directory under your test name e.g. `store/onyx-basic/20160118T102259.000Z`. You can view a sample of onyx-jepsen's [result.edn](https://gist.github.com/lbradstreet/60c4be48216146878f58). In addition to the standard Jepsen output, we also copy Onyx's log output to the test run directory. Scientists often like to keep a log of experimental results, and we have tried to emulate one further, keeping a log of our immediate interpretations and hypothesis, of each failed run. See this [sample if you are interested in our process](https://github.com/onyx-platform/onyx-jepsen/blob/master/onyx-issues-log.txt#L233), but please do not judge our notes! 

Onyx coordinates peers via a shared log, written to ZooKeeper. Each peer plays back this log in order, gaining a full view of the cluster replica. One advantage of this mechanism is that we can playback the log obtained by jepsen, and debug it step by step. A great post [describing this design pattern](https://news.ycombinator.com/item?id=10765378) has been written by [Brandon Bloom](https://twitter.com/BrandonBloom). It is this pattern that makes testing our replica coordination, and cluster scheduler easy with property testing, and is now paying dividends with our Jepsen testing.

To this end, we wrote a [console application](https://github.com/onyx-platform/onyx-console-dashboard) that opens Jepsen's [result.edn](https://gist.github.com/lbradstreet/60c4be48216146878f58) outputs, allowing us to step through the replica, diff each action, filter by peer actions, ids, etc. This vastly simplifies debugging coordination and scheduler related issues.

<img src="{{ '/assets/jepsen_viz/console_dashboard.png' | prepend: site.baseurl }}" height="70%" width="70%">

### Testing Onyx's State Management feature

The previous Onyx test was a test of our cluster fault tolerance mechanisms and scheduler. In our next test, we will stress our [windowing](https://github.com/onyx-platform/onyx/blob/0.8.x/doc/user-guide/windowing.md) and [state management](https://github.com/onyx-platform/onyx/blob/0.8.x/doc/user-guide/aggregation-state-management.md) features, which are intrinsically linked by the way they incrementally journal aggregation state machine updates, window results, and transactional [exactly once aggregation updates](https://github.com/onyx-platform/onyx/blob/0.8.x/doc/user-guide/aggregation-state-management.md#exactly-once-aggregation-updates) (not to be confused with exactly once side effects, which are impossible).

Onyx's state management and windowing features journal each state update and corresponding unique id, to BookKeeper. Upon the failure of a peer, the state machine log is replayed to recover the full state. In this next test, we build a job that adds each message to a collection, using the `:onyx.windowing.aggregation/conj` aggregation, over a window on the `:annotate-job` task. Onyx's "exactly once" / de-duplication feature, will ensure that this message will only be added to this collection once. Once all messages are processed, the final state must consist of all of the messages written to all of the ledgers by the Jepsen clients.

*The window on the `:annotate-job` task:*

```clojure
{:window/id :collect-segments,
 :window/task :annotate-job,
 :window/type :global,
 :window/aggregation :onyx.windowing.aggregation/conj,
 :window/window-key :event-time}
```

In order to check this final state, we also add a trigger to the window above. This trigger is configured to persist the full window state to BookKeeper, and only writes when a peer is stopped. The Jepsen checker reads the result of the the final trigger call, and checks it against the data written by the clients to the input BookKeeper ledgers. All data must be available in the final write, but must not be occur more than once, as that would violate de-duplication.

*The trigger on the `:collect-segments` window:*

```clojure
{:trigger/window-id :collect-segments,
 :trigger/refinement :onyx.triggers.refinements/accumulating,
 :trigger/on :onyx.triggers.triggers/segment,
 :trigger/threshold [1 :elements],
 :trigger/sync :onyx-peers.functions.functions/update-state-log}],
```

*The Onyx Job DAG (hover to view task data):*

<iframe src="{{ '/assets/jepsen_viz/stateful.html' | prepend: site.baseurl }}" width="960" height="340" scrolling="no"></iframe>

This test found two issues that were previously known by the Onyx team, but were theoretical as they had not been seen in practice.

* [BookKeeper state log / key filter interaction issue #382](https://github.com/onyx-platform/onyx/issues/382) 
* [Failed async BookKeeper writes should cause peer to to restart #390](https://github.com/onyx-platform/onyx/issues/390)

Jepsen was a powerful ally in fixing these bugs as it gave us certainty that we had fixed them correctly. We have internally joked about this as JDD (Jepsen Driven Development).

### Kill -9 Me

One of our production users reported an issue where a cluster had troubles recovering from a full cluster restart. We copied a [crash nemesis](https://github.com/onyx-platform/onyx-jepsen/blob/master/src/onyx_jepsen/onyx_test.clj#L123) from Jepsen's [elasticsearch](https://github.com/aphyr/jepsen/blob/master/elasticsearch/src/elasticsearch/core.clj) tests. This nemesis kill -9s between 1-5 of the Jepsen nodes in each nemesis event. We re-used the simple job tested in our first test setup.  When all 5 of the nodes are killed, Jepsen reproduced the issue reported by our user. After reviewing the peer logs using the console dashboard, we were able to quickly discover the source of the issue and [provide a fix](https://github.com/onyx-platform/onyx/pull/526) that we had confidence in.

In the future we will use a similar kill -9 test against our stateful jobs, and against BookKeeper, to test for full recovery of task state.

### Things we Learned

Test your assumptions. We had not realized that BookKeeper would commit suicide upon losing quorum. This is a good practice whether you are building a distributed system or not.

Building tests with Jepsen can take a long time and has a bit of a learning curve, however it is incredibly worthwhile. This type of testing has greatly increased confidence in our product, and proven helpful in reproducing issues seen in the wild that would otherwise be difficult. Jepsen will also provide further confidence in refactoring our code, including building other forms of fault tolerance into our system.

Store your application's logs upon test completion. Obviously it is very helpful to be able to correlate the info and exceptions logged by your application with the events generated by Jepsen.

Feedback loops while developing Jepsen tests can be long. We improved turn around times by pre-building docker images with all of our dependencies installed. 

We further improved test development time by building a test harness ([example](https://github.com/onyx-platform/onyx-jepsen/blob/master/test/onyx_peers/jobs/basic_test.clj#L23)) around Jepsen and Onyx, using with static generated events that use a single client, and no nemesis. These tests spin up a development mode Onyx cluster in the JVM without Jepsen orchestrating nodes being spun up and destroyed. This allowed us to build new tests quickly and re-factor our tests as required. We then use substantially similar tests with Jepsen orchestrating real nodes, a nemesis, and generated events. It also allowed us to CI test our Jepsen tests.

### The Future

We will continue to add tests to [onyx-jepsen](https://github.com/onyx-platform/onyx-jepsen/). Next up is further testing around fault tolerance aspects in triggers, aggregation grouping, and more. Following the lead of the Call Me Maybe posts, we also want to test performance and recovery characteristics resulting from nemesis events.

Jepsen testing should also be integrated into our CI process. As users of [CircleCI](http://www.circleci.com/), it is difficult for us to do this testing directly on Circle, as we will quickly hit resource limits. We are considering having successful CI builds trigger starting a spot instance on EC2 that runs our Jepsen suite.

The test harness described above, may give Onyx a path to building all of our integration tests in a way that we can easily reuse them with Jepsen. This would require some re-factoring of our tests, primarily to be built around generators, however there are no technical obstacles standing in our way. 

Taking this idea even further, Onyx may be able to create testing functionality that essentially provides Jepsen testing of end user's jobs for free. The main obstacle here is providing a way to allow plugin projects to be spun up outside of the purview of the Jepsen nemesis. As we are currently using docker-in-docker to run our Jepsen tests, this may be easy to provide. If we are able to achieve this in a sane, and easy to use manner, this would be a feature provided by no other solution that we know of.

### To Hear More & Our Pitch

If you are interested in hearing further thoughts on Onyx and distributed systems, please subscribe to [Distributed Masonry's Newsletter](http://eepurl.com/beFW_P) and follow us [on Twitter](http://www.twitter.com/onyxplatform).

Distributed Masonry are also [available](http://www.onyxplatform.org/support/) for Onyx Platform and general distributed systems consulting, contracting, support, and training services. Please feel free to contact us if you are interested in our services or just want to have a [chat about this post](mailto:support@onyxplatform.org).

### Thanks

Thank you to Kyle Kingsbury, Michael Drogalis, Bridget Hillyer, and Gardner Vickers for reviewing this post.

-- Distributed Masonry, [Lucas Bradstreet](http://www.twitter.com/ghaz)

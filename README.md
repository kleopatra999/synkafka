# SynKafka: A simple, synchronous C++ producer client API for Apache Kafka =

### You almost certainly should not use this.

If you want a C/C++ Kafka client use [librdkafka](https://github.com/edenhill/librdkafka).

This library is **incomplete** and potentially **slow**.

It's purpose is as a low-level, simplistic implementation of the Kafka 0.8 producer protocol which allows more control over how they produce and their transactional/error handling semantics.

## Rationale

The motivating case is wanting to write a [Facebook Scribe](https://github.com/facebookarchive/scribe) Store that writes to Kafka cluster in a similar way to how NetworkStore writes to an upstream Scribe server. In particular, the common case of using a BufferStore to write to upstream until upstream fails, spool to disk until upstream is back, replay from disk once upstream is back and finally resume live sending.

This is very hard to achieve with an asynchronous API like librdkafka where you have no visibility into upstream node's availability, and can only handle failures per-message rather than reason about a whole partition's availability in general.

Since Scribe already handles batching and partitioning into topics (and potentially partitions depending on how you configure it),
we needed a much lower level API where we can make our own trade-offs about batching efficiency vs. simplicity in error handling.

For example we might choose to only produce to a single partition in any produce request such that if that partition is unavailable
we can fail the KafkaStore and cause a BufferStore to switch to DISCONECTED (disk-spooling) state. Limiting batches to single partition potentially reduces throughput where you have many topics/partitions, but makes Scribe's BufferStore online/offline model sane to work with.

We do take care to take advantage of Kafka's pipelining though such that multiple stores (assuming they are configured to run in separate threads) can be sending batches to same broker in parallel, with only a single broker connection held open by the client.

## Limitations

 * Only some of the API is implemented currently - only enough to discover where partitions are and produce to them.
 * API is low-level and requires external work (queuing, multi-threading) to get good performance.
 * We did not build this with performance as a primary concern. That said it doesn't have too many pathological design choices. In trivial functional tests running on Quad core/16GB laptop with dockerised 3 node kafka cluster on boot2docker VM, with batches of 1000 ~60 byte messages, we see sending rates of 100-200k messages a second on aggregate across 8 sending threads. That is without any tuning of messages/thread count let alone proper profiling of code. It well exceeds our current requirements so performance has not been optimized further.

## Dependencies

 - boost (tested with 1.58, somewhat older will probably work)
 - boost build
 - recentish gcc (tested with 4.9.2) (uses `-std=c++11`)
 - probably other things to have working build chain

This library aims to build statically with vendored dependencies.

That said some dependencies were more trouble than they are worth to vendor and build as part of project's build system so
we currently assume the following are installed in system search paths:

 - zlib (tested with 1.2.8, linked statically)
 - boost::asio (headers only but expected in system include path)
 - boost::system (linked statically)

### OS X

Use [homebrew](http://brew.sh/) to install deps:

`brew install boost boost-build zlib`

## Building

Clone repo and cd into repo root. From project root dir run:

`b2 release` or just `b2` to build with debugging symbols.

## Running Tests

To build and run unit test use:

`b2 test` or `b2 release test`

To run the included functional tests against a local kafka cluster, look at the notes in `./tests/functional/setup_cluster.sh`. This should be run from project root once pre-requisites documented there are met.

This includes `docker-compose` config to build and configure a 3 node kafka cluster with a single zookeeper node.

The setup script can be re-run to automatically reset your local cluster to pristine state.

It must be run before each functional testing session, although individual tests should clean up after themselves so it's not normally required to run it between test runs unless a test aborts and leaves cluster broken or similar.

Once cluster is up, you can run functional tests with:

`./tests/functional/run.py`

This should be run from the project root and will re-compile debug build automatically if any changes were found.

Runner uses gtest so you can pass normal gtest command options to it, for example:

`./tests/functional/run.py --gtest_filter=ProducerClientTest.*`

All tests pass individually however some functional tests that intentionally cause cluster failures do not recover in a deterministic time so despite some generous `sleep()` calls in the tests, can cause subsequent test to fail spuriously when running entire suite.

## Usage

The public API is intentionally simple, with essentially 2 useful methods and some configuration options.

Simple example:

```c++

auto client = synkafka::ProducerClient("broker1,broker2:9093");

// call client.set_* to configure parameters if defaults are not suitable (see synkafka.h for details)

// Blocking call - checks if topic/partition is known, and the leader can have TCP connection opened
// If not it returns non-zero std::error_code. See synkafka.h for more details.
// This is thread safe and many threads can call it even with same topic/partition arguments concurrently.
auto ec = client.check_topic_partition_leader_available("topic_name", 0);

if (ec) {
	// partition leader not available
}

// Produce a batch of messages
synkafka::MessageSet messages;

// Optionally set compression type, max_message_size config from your Kafka setup (if not using default)
// with messages.set_*

for (std::string& message : your_message_list) {
	auto errc = messages.push(message, /* optional key */ "");
	if (errc == synkafka::synkafka_error::message_set_full) {
		// The message set is full according to Kafka's max_message_size limits
		// you probably want to send it now and continue with you messages in another batch after that
	} else if (errc) {
		// Wild zombies attacked (or some other equally unexpected problem like bad_alloc)
	}
}

// Actually send it. This blocks until we either sent, got an error, or timed out.
// inspect ec to decide which...
ec = client.produce("topic_name", /* partition id = */ 0, messages);

```
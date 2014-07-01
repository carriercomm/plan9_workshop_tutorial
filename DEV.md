# Add your own Badge to the Plan9 NodeJS Backend

For more information on installing all pre-requistes and playing Plan 9 From Outer Space, see the main [README.md](README.md).


## Introduction

The Plan9 From Outer Space game demonstrates the integration of a massively parallel data-driven real-time backend based on Apache Storm into a web application that features frequent user interaction. For demonstration purposes, we use the Storm backend to notice certain events, based on which users are awarded "badges". Badges modify the number of points of a user (awarding bonus points, removing penalty points)or trigger specific actions (like giving a user the superpower of receiving the points generated by all other users for a certain period of time). 

These simple badges, however, demonstrate problemsoften encoutered in the real-world, like performing joins of multiple data streams, or enhancing user events with background informations (like noticing when a user enters the website category, where he spends most of his time and awarding special offers based onthe revenue he generated last month).

The [HighFiveStreamJoinBolt]() provides a simple example of joining tuples in a data stream using the Redis key-value store. 

Here, we will create another Bolt that performs a local aggregation (for simplicity also backed by Redis) and periodically emits a data tuple based on that short-lived aggregation.

The story background in the Plan9 game is as follows:

- Users play and uncover pixels, earning points
- Periodically, zombies called the "Kitten Robbers From Outer Space" will come and steal kittens from some player.
- Because the zombies are deluded in their quest to equalize the world, they will always attack the player that recently made the most points

The Storm bolt thus has to implement the following logic:

- Read the Plan9 points tuple stream
- Aggregate the generated points for each user 
- After a certain interval, pick the user with the most points and create a badge message that:
	1. Informs the user of the attack
	2. Remove 20% of the recently earned points from the user to simulate a kitten robbery
- Clear the aggregation map for the next round


To trigger the periodic attack action, we will use a Storm feature called "tick tuples". Each component in a Storm topology can be individually configured to receive a "tick tuple" from Storm itself at a pre-set frequency. We provide a skeleton Node.js bolt implementation that processes tick tuples: [EmptyTickTupleBolt.js](src/plan9/bolts/EmptyTickTupleBolt.js), as well as a Storm multilang adapter that provides an easy way to configure tick tuples: [MultilangAdapterTickTupleBolt.java](deck36-storm-backend-nodejs/blob/master/src/jvm/deck36/storm/general/bolt/MultilangAdapterTickTupleBolt.java).

Read more about Storm tick tuples here: [Excursus: Tick Tuples in Storm 0.8+](http://www.michael-noll.com/blog/2013/01/18/implementing-real-time-trending-topics-in-storm/#excursus-tick-tuples-in-storm-08).


## Implement NodeJS Component

To create the described Diluted Kitten Robbers Features, we will first create a new Node.js bolt called "DelutedKittenRobbersBolt" based on the [EmptyTickTupleBolt.js](src/plan9/bolts/EmptyTickTupleBolt.js) in the "deck36-api-backend" project:
	
	cd src/plan9/bolts/
	cp EmptyTickTupleBolt.js DeludedKittenRobbersBolt.js


Similar to our [HighFiveStreamJoinBolt](), we need the *redis* and the *fibers/future* (more on that later) libraries and we need to get a redis client:

	var nodeRedis = require('redis');
	var Future = require('fibers/future');

	//The Bolt constructor takes a definition function,
	//an input stream and an output stream
	var joinBolt = new Bolt(function(events) {
	    var collector = null;
	    var redis = null;

	    //You can listen to the bolt "prepare" event to
	    //fetch a reference to the OutputCollector instance
	    events.on('prepare', function(c) {
	        collector = c;
	        redis = nodeRedis.createClient();
	    });



In the *Bolt* callback, we find this part:

	if (isTickTuple(tuple)) {
        this._protocol.sendLog("\n\n\nTICK TUPLE FROM NODE.JS: " + JSON.stringify(tuple) + "\n\n");
    } else {
        this._protocol.sendLog("\n\n\nTUPLE FROM NODE.JS: " + JSON.stringify(tuple) + "\n\n");
    } 

First, we create two functions, one to process a real tuple and one to process a tick tuple. Both functions are provided with the "_protocol" instance from the bolt (contains the storm configuration) and the actual tuple:

	var processTick = function(protocol, tuple) {
		// ..
	}

	var processTuple = function(protocol, tuple) {
		// ..
	}

We now need to call these functions in our main *Bolt* callback. Because we will interact with a redis store and want to avoid callback hell, we need some method to deal with asychronous database queries. To this end, we use the the [node-fibers](https://github.com/laverdet/node-fibers/) extension. In order to use its magic later, we need to wrap our main *Bolt* callback into a *Fiber*. Because we onyl use the more high-level *Futures* abstraction, we can use an easy ".future()" wrapper provided by node-fiber. Our main *Bolt* callback thus becomes:


	return function(tuple, cb) {

        if (isTickTuple(tuple)) {
            processTick(this._protocol, tuple);
        } else {
            processTuple(this._protocol, tuple);
        }

        collector.ack(tuple);
        cb();
    }.future()

Looks simple enough. 


Now we need to add our business logic to *processTuple* and *processTick*. 

Our *processTuple* function first checks again, whether we actually did get a *points* message:

	var object = tuple["values"][0];
    var type = object["type"];

    if (type != "points") {
        collector.ack(tuple);
        cb();
        return;
    }

If we get a non-"points" message, we simply ack it ad move on...

If we actually got one, we simply extract the user id and her recent points and increment a counter in [redis sorted set](http://redis.io/topics/data-types#sorted-sets):
	
	// get user id
	var user = object['user_target']['user_id'];
   
    // get points increment
    var pointsIncrement = object["points"]["increment"];

    // update point counter in redis zset
    redis.zincrby("plan9:ExtendedKittenRobbersZSet", parseInt(pointsIncrement), user);

For this demo case, we deliberately ignore the possibility of incrementing the counter multiple times (because in failure cases, Storm will replay tuples). An interesting side note is, that the question of recent activity (Did the user produce points? vs. How many points did the user produce?) is *idempotent*, i.e. the answer will never change no matter how often we increment. In that case, our deluded kitten robbers will simply hyperventilate over our players and their kittens. For cases where overcounting is not an option, multiple strategies exist depending on actual requirements. But that discussion is beyond our scope here. 


Now the last step is to implement the actual attack in *processTick*. First, we need to query our redis sorted set for the top user. Using node-fiber futures, that is a piece of cake:

	var userFuture = new Future;

    redis.zrevrange("plan9:ExtendedKittenRobbersZSet", 0, 0, function(err, reply) {

        if (err) {
            collector.fail(tuple);
            cb();
            userFuture.return ({});
        }

        userFuture.return (reply);
    });

    var winner = userFuture.wait();

If we should have a winner (maybe nobody made a move in the last round, because their minds were still stunned by the Kitten Robbers), we need to query her points as well:

	if (winner.length > 0) {
        // get aggregated points of the 'winning' user
        var scoreFuture = new Future;

        redis.zscore("plan9:ExtendedKittenRobbersZSet", winner, function(err, reply) {

            if (err) {
                collector.fail(tuple);
                cb();
                scoreFuture.
                return ({});
            }

            scoreFuture.
            return (reply);
        });

        var winnerScore = scoreFuture.wait();

The *winner* (LOL) will now loose about 20% of the points she made in the last round (*har har*):

	var penaltyScore = Math.round(-1 * winnerScore * 0.20);

    protocol.sendLog("\n\n\nEXTENDED KITTER ROBBER WILL ATTACK USER " + JSON.stringify(winner) + "  -- SCORE: " + winnerScore + "  -- PENALTY: " + penaltyScore + "\n\n");


Now that we have found our winner, we need to clear our set for the next round:

    // clear scores for next round
    redis.del("plan9:ExtendedKittenRobbersZSet");

.. and build a "badge" message, we can then send back to the game:

    // build attack badge
    var kittenRobbersFromOuterSpace = {};

    // get storm config 
    var stormConf = protocol.getStormConf();

    kittenRobbersFromOuterSpace['badge'] = {};
    kittenRobbersFromOuterSpace['badge']['name'] = stormConf['deck36_storm']['ExtendedKittenRobbersBolt']['badge']['name'];
    kittenRobbersFromOuterSpace['badge']['text'] = stormConf['deck36_storm']['ExtendedKittenRobbersBolt']['badge']['text'];
    kittenRobbersFromOuterSpace['badge']['size'] = stormConf['deck36_storm']['ExtendedKittenRobbersBolt']['badge']['size'];
    kittenRobbersFromOuterSpace['badge']['color'] = stormConf['deck36_storm']['ExtendedKittenRobbersBolt']['badge']['color'];
    kittenRobbersFromOuterSpace['badge']['effect'] = stormConf['deck36_storm']['ExtendedKittenRobbersBolt']['badge']['effect'];

    kittenRobbersFromOuterSpace['points'] = {};
    kittenRobbersFromOuterSpace['points']['increment'] = penaltyScore;

    kittenRobbersFromOuterSpace['action'] = {};
    kittenRobbersFromOuterSpace['action']['type'] = 'none';
    kittenRobbersFromOuterSpace['action']['amount'] = 0;

    kittenRobbersFromOuterSpace['type'] = 'badge';
    kittenRobbersFromOuterSpace['version'] = 1;
    kittenRobbersFromOuterSpace['timestamp'] = new Date().getTime();

    // specify the user 
    kittenRobbersFromOuterSpace['user'] = {};
    kittenRobbersFromOuterSpace['user']['user_id'] = parseInt(winner[0]);

    // and log for the fun of it...
    protocol.sendLog("\n\n\nEXTENDED KITTER ROBBER BADGE " + JSON.stringify(kittenRobbersFromOuterSpace) + "\n\n");


Once we have build our message, we now just need to emit the message. Not, that we need to emit an array as our tuple. Thus `[kittenRobbersFromOuterSpace]` translates to a tuple of size 1, where the first entry is our badge object. The "emit" method accepts the to-be-emitted tuple, a target stream to emit to, and anchors. Anchors are simply other tuples, that the emitted tuple is anchored to. By doing so, a "tuple tree" (actually, a graph) is created. This "tuple tree" is used by Storm to track and eventually re-play certain tuples. Only anchored tuples need to be ack- or fail'ed. Because our bolt emits based on tick tuples, we don't anchor them here.

    // emit the badge
    collector.emit([kittenRobbersFromOuterSpace]); 

And that's it already!



## Create Plan9 config for DeludedKittenRobbersBolt

Similar to the other bolts, we mainly configure the name of the script to run in `storm.yml`:

	DeludedKittenRobbersBolt:
        main:				"/home/vagrant/plan9/deck36-api-backend/src/plan9/bolts/DeludedKittenRobbersBolt.js"
        params:
        rabbitmq:
            exchange:        "plan9"
            routing_key:     "points.#"
            target_exchange: "plan9-backchannel"
        robber_frequency:    10
        badge:
            name:   "KittenRobbersFromOuterSpace"
            text:   "The mysterious Kitten Robbers came from Outer Space and took some of your kitties away! Oh no!!!!1"
            color:   "#232312"
            size:    "30em"
            effect:  "puff" 



To execute our production version script when deployed to a cluster, we simply overwrite `DeludedKittenRobbersBolt.main` in `storm_prod.yml`:

	DeludedKittenRobbersBolt:
        main:       "DeludedKittenRobbersBolt.prod.js"





## Implement Storm Topology 


We can now use our shiny new Node.js bolt in any Storm topology using our [MultilangAdapterTickTupleBolt.java](deck36-storm-backend-nodejs/blob/master/src/jvm/deck36/storm/general/bolt/MultilangAdapterTickTupleBolt.java) wrapper. However, for our game, we will now create a single topology that only employs our new DeludedKittenRobbersBolt.

To create that topology, we start from the *HighFiveStreamJoinTopology* already present in our "deck36-storm-backend-nodejs" project:

	cd src/jvm/deck36/storm/plan9/nodejs/
	cp HighFiveStreamJoinTopology.java DeludedKittenRobbersTopology.java

Let's now walk through this topology an make the necessary changes.

First, let's rename our topology:

	public class DeludedKittenRobbersTopology {

	    private static final Logger log = LoggerFactory.getLogger(DeludedKittenRobbersTopology.class);


Then, we already come to our *main* method. The first part check, if we have supplied a command line parameter that must be either *"dev"* or *"prod"*.   

	    public static void main(String[] args) throws Exception {

	        String env = null;

	        if (args != null && args.length > 0) {
	            env = args[0];
	        }

	        if (! "dev".equals(env))
	            if (! "prod".equals(env)) {
	                System.out.println("Usage: $0 (dev|prod)\n");
	                System.exit(1);
	            }

We will use that parameter for:
	
1. Choose which config we will load.
2. Decide whether to run the topology in either local mode ("dev") or whether it should be deployed to the cluster ("prod").

Storm features a topology-wide config. This config controls several aspects of Storm internals (like timeouts, etc.), and is also forwarded into each component (bolts or spouts). We will thus add our application config to the main topology config. We will load our application config from YaML files supplied with the "deck36-storm-backend-nodejs" project, but we could also integrate this config with our main Node.js project. if you're interested in integration of configs, check our PHP tutorial [deck36-plan9-storm-tutorial-php]() where we read the config from the main PHP project and use PHP bolts to update data entities defined in the main PHP project. But for now, let's just read our config:

	        // Topology config
	        Config conf = new Config();

	        // Load parameters and add them to the Config
	        Map configMap = YamlLoader.loadYamlFromResource("storm_" + env + ".yml");

	        conf.putAll(configMap);

	        log.info(JSONValue.toJSONString((conf)));

	        // Set topology loglevel to DEBUG
	        conf.put(Config.TOPOLOGY_DEBUG, JsonPath.read(conf, "$.deck36_storm.debug"));



The next main part is the Storm TopologyBuilder. The TopologyBuilder is used to add all components (bolts/spouts) and to configure all of their settings (connections between components, parallelism).

	        // Create Topology builder
	        TopologyBuilder builder = new TopologyBuilder();

There are two parameters for configuring the parallelism. The *parallelism hint* configures the number of *executors*, while *num tasks* configures the number of *tasks*. An *executor* is an actual JVM thread that reads and writes from/to the internal Storm queues. Two separate *executors* can thus be run on two different machines. A *task*, however, is a pseudo-parallelization within one *executor*. If you have, for instance, two *executors* and 8 *tasks, then Storm will launch two threads executing *4* tasks each. Within one *executor*, these tasks will be executed sequentially. 

This starts to make sense taking two moe points into account: the number of *tasks* is fixed for the lifetime of a topology, while the number of actual *executors* can be changed dynamically. Thus, by initially specifying just one *executor*, but 8 *tasks* you can later on expand processing for that component onto up to 8 machines. 

While *tasks* are very lightweight as long as you use pure JVM components, one caveat when using the multilang protocol is that each *task* spawns its own external processing process. Thus, a JVM bolt with one *executor* and 8 *tasks* will equal to one thread in the JVM, while a multilang bolt with the same configuration will be equal to one JVM thread plus 8 spawned system processes. Therefore, don't overplan when using multilang bolts. As a side note, there are further strategies, like deploying a certain topology multiple times or do rolling redeployments, to deal with dynamic parallelism changes that might be more suitable when dealing with a lot multilang bolts. 

	        // if there are not special reasons, start with parallelism hint of 1
	        // and multiple tasks. By that, you can scale dynamically later on.
	        int parallelism_hint = JsonPath.read(conf, "$.deck36_storm.default_parallelism_hint");
	        int num_tasks = JsonPath.read(conf, "$.deck36_storm.default_num_tasks");


Now, having finished preparing our configuration, we come to our first component: a spout reading messages from RabbitMQ from [storm-rabbitmq](https://github.com/ppat/storm-rabbitmq). Please check the repository from that poject for more information on how to use and configure the spout. The main thing we have to do here and now is to change the JSONPath to our bolt configuration. We will use the key "DeludedKittenRobbersBolt" for this configuration, which we will create later:

	        // Create Stream from RabbitMQ messages
	        // bind new queue with name of the topology
	        // to the main plan9 exchange (from properties config)
	        // consuming only CBT-related events by using the rounting key 'cbt.#'

	        String badgeName = DeludedKittenRobbersTopology.class.getSimpleName();

	        String rabbitQueueName = badgeName; // use topology class name as name for the queue
	        String rabbitExchangeName = JsonPath.read(conf, "$.deck36_storm.DeludedKittenRobbersBolt.rabbitmq.exchange");
	        String rabbitRoutingKey = JsonPath.read(conf, "$.deck36_storm.DeludedKittenRobbersBolt.rabbitmq.routing_key");


	        // Get JSON deserialization scheme
	        Scheme rabbitScheme = new SimpleJSONScheme();

	        // Setup a Declarator to configure exchange/queue/routing key
	        RabbitMQDeclarator rabbitDeclarator = new RabbitMQDeclarator(rabbitExchangeName, rabbitQueueName, rabbitRoutingKey);

	        // Create Configuration for the Spout
	        ConnectionConfig connectionConfig =
	                new ConnectionConfig(
	                        (String)    JsonPath.read(conf, "$.deck36_storm.rabbitmq.host"),
	                        (Integer)   JsonPath.read(conf, "$.deck36_storm.rabbitmq.port"),
	                        (String)    JsonPath.read(conf, "$.deck36_storm.rabbitmq.user"),
	                        (String)    JsonPath.read(conf, "$.deck36_storm.rabbitmq.pass"),
	                        (String)    JsonPath.read(conf, "$.deck36_storm.rabbitmq.vhost"),
	                        (Integer)   JsonPath.read(conf, "$.deck36_storm.rabbitmq.heartbeat"));

	        ConsumerConfig spoutConfig = new ConsumerConfigBuilder().connection(connectionConfig)
	                .queue(rabbitQueueName)
	                .prefetch((Integer) JsonPath.read(conf, "$.deck36_storm.rabbitmq.prefetch"))
	                .requeueOnFail()
	                .build();

	        // add global parameters to topology config - the RabbitMQSpout will read them from there
	        conf.putAll(spoutConfig.asMap());

	        // For production, set the spout pending value to the same value as the RabbitMQ pre-fetch
	        // see: https://github.com/ppat/storm-rabbitmq/blob/master/README.md
	        if ("prod".equals(env)) {
	            conf.put(Config.TOPOLOGY_MAX_SPOUT_PENDING, (Integer) JsonPath.read(conf, "$.deck36_storm.rabbitmq.prefetch"));
	        }


We can now add our RabbitMQ spout by using the "setSpout()" method of our TopologyBuilder instance:

	        // Add RabbitMQ spout to topology
	        builder.setSpout("incoming",
	                new RabbitMQSpout(rabbitScheme, rabbitDeclarator),
	                parallelism_hint)
	                .setNumTasks((Integer) JsonPath.read(conf, "$.deck36_storm.rabbitmq.spout_tasks"));



The messages emitted by the RabbitMQ spout are then directly fed into our Node.js bolt. In order to invoke it, we need to build a list representing our invocation of the "node" binary, the javascript file and optional arguments. 

	        // construct command to invoke the external bolt implementation
	        ArrayList<String> command = new ArrayList(15);

	        // Add main execution program 
	        command.add((String) JsonPath.read(conf, "$.deck36_storm.nodejs.executor"));

	        // Add main route to be invoked and its parameters
	        command.add((String) JsonPath.read(conf, "$.deck36_storm.DeludedKittenRobbersBolt.main"));
	        List boltParams = (List<String>) JsonPath.read(conf, "$.deck36_storm.DeludedKittenRobbersBolt.params");
	        if (boltParams != null)
	            command.addAll(boltParams);

	        // Log the final command
	        log.info("Command to start bolt for Deluded Kitten Robbers: " + Arrays.toString(command.toArray()));


We are now ready to actually add our Node.js bolt to the topology. To this end, we use the [MultilangAdapterTickTupleBolt.java](deck36-storm-backend-nodejs/blob/master/src/jvm/deck36/storm/general/bolt/MultilangAdapterTickTupleBolt.java) and read the tick frequency from the configuration. We simply call our bolt "badge" and connect to the RabbitMQ spout by using `.shuffleGrouping("incoming")`. Note: "incoming" is the name we have given to our RabbitMQ spout. 

			// Add constructed external bolt command to topology using MultilangAdapterTickTupleBolt
			builder.setBolt("badge",
			        new MultilangAdapterTickTupleBolt(
			                command,
			                (Integer) JsonPath.read(conf, "$.deck36_storm.DeludedKittenRobbersBolt.robber_frequency"), 
			                "badge"
			        ),
			        parallelism_hint)
			        .setNumTasks(num_tasks)
			        .shuffleGrouping("incoming");


We will now receive the Kitten Robber attack badges from our `badge` bolt. We now only need two more bolts, one to serialize and add routing informations, and another one that actually pushes the data abck to RabbitMQ. The router bolt and the push bolt could be easily merged into one, but the separation keeps the push bolt so general, that it can be easily used to push messages received from multiple Storm streams within more complex topologies:

	        builder.setBolt("rabbitmq_router",
	                new Plan9RabbitMQRouterBolt(
	                        (String) JsonPath.read(conf, "$.deck36_storm.DeludedKittenRobbersBolt.rabbitmq.target_exchange"),
	                        "DeludedKittenRobbers" // RabbitMQ routing key
	                ),
	                parallelism_hint)
	                .setNumTasks(num_tasks)
	                .shuffleGrouping("badge");

	        builder.setBolt("rabbitmq_producer",
	                new Plan9RabbitMQPushBolt(),
	                parallelism_hint)
	                .setNumTasks(num_tasks)
	                .shuffleGrouping("rabbitmq_router");


Hooray! Our topology is finished and be executed! For testing purposes, a Storm cluster can be emulated using *LocalCluster*. In that case, we need to add an artificial delay using `Thread.sleep()` to keep the JVM from exiting after submitting the topology to the *LocalCluster* in the same JVM.  

	        if ("dev".equals(env)) {
	            LocalCluster cluster = new LocalCluster();
	            cluster.submitTopology(badgeName + System.currentTimeMillis(), conf, builder.createTopology());
	            Thread.sleep(2000000);
	        }

For production use, we only need to use the static `StormSubmitter.submitTopology()' and off we go! 

	        if ("prod".equals(env)) {
	            StormSubmitter.submitTopology(badgeName + "-" + System.currentTimeMillis(), conf, builder.createTopology());
	        }

Et voila!







## Package Storm Topology

1. We need to wrap up our Node.js bolt in our deck36-api-backend project. We use [sardines]() to wrap all javascript files into our final file. However, because fibers uses a native binary that can't be wrapped into the file, we need to fix the directory, where the module is required. To this end, we provide the [pathLocalToGlobalFiber.sh] shell script.  

	# create prod bolt in target directory
	
	cd target/
	sardines ../src/plan9/bolts/DeludedKittenRobbersBolt.js -o DeludedKittenRobbers.prod.js
	../bin/patchLocalToGlobalFiber.sh DeludedKittenRobbers.prod.js


2. We need to compile our Java wrappers and the Java Topology and bundle it all up into an "uberjar" that contains all dependencies and can be deployed to the cluster. Note: As of July, 2014, Storm and all submitted topologies will share the same classpath. That might cause problems, when requiring dependencies in different incompatible version to those that are required by Storm. There are efforts to change that in the future, but it hasn't yet arrived. However, as we use multilang bolts fo our main logic, we don't need to care too much. For the same reason, we need Storm as a dependency to compile our classes, but we must not include it in our uberjar. Otherwise it will not be possible to submit the topology to an actual cluster. Our `build.sh` handles that for us. 

The `target` directory of our "deck36-api-backend" project (with our production-ready Node.js script) is liked to our "deck36-storm-backend-nodejs" and will thus be included in the JAR file automatically.

	./build.sh


## Run 

### Locally (for testing)

In our vagrant enviroment, Storm is installed in `/opt/storm`. We can thus execute the topology locally by:

	/opt/storm/bin/storm jar target/deck36-storm-backend-nodejs-0.0.1-SNAPSHOT-standalone.jar deck36.storm.plan9.nodejs.ExtendedKittenRobbersTopology dev

Note: To run the topology in dev mode within the vagrant, make sure your node binary and javascript paths in the yaml files are correct and don't differ due to your local development machine setup.


### Deploy to Cluster 

Deploying to a cluster is very simple as well, because all the cluster config is handled by the `storm` command. We thus just need to switch `dev` to `prod`:

	/opt/storm/bin/storm jar target/deck36-storm-backend-nodejs-0.0.1-SNAPSHOT-standalone.jar deck36.storm.plan9.nodejs.ExtendedKittenRobbersTopology prod



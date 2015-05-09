# Twitter Updater Specification.

## Contents.

* [1. Introduction](#introduction)
* [2. Rationale](#rationale)
* [3. Problem](#problem)
* [4. Constraints](#constraints)
* [5. Background](#background)
* [6. Hypothesis](#hypothesis)
* [7. Assumptions](#assumptions)
* [8. Solution](#solution)
	* [8.1. Blackbox](#blackbox)
	* [8.2. Theory of Operation](#toop)
* [9. Technichal Specofication](#tech)
	

<h2 id="introduction">Introduction.</h2>

Tweets, just like living creatures have a life time, even when they don't get deleted they practically die because the older they get the less people interact with them until no one ever sees them, retweets them or interacts with them in any way. This also means that a young tweet has a better chance to get interactions: It will show in feeds giving the oportunity to be retweeted, favorited, clicked, etc.

Some software applications can benefit from getting this interaction information in a fast and reliable way. It can be that your software is intended to pull accurate stats from a twitter account, or a specific sets of tweets, hashtags, mentions, links that got shared, a keyword, etc.

This is the first version of this specification. It is meant to be used for keeping tweets updated in your system, the analysis algorithms and heuristics you aplly to it depend entirely on the problem you are solving.

<h2 id="rationale">Rationale.</h2>

As mentioned before, we can benefit from knowing about the interactions with tweets and the retweets they produce. It tells us the results of particular tweets and there are many ways to monetize this information.

This is a simple and powerful approach that focuses on getting updated on-demand information about the tweets you are interested in, it is meant to be easily scaled up, simply by adding more instances of a process to the mixture.

> Simplicity is prerequisite for reliability. - Edsger Dijkstra

<h2 id="problem">Problem.</h2>

Keep every tweet in a system updated according to their age regardless of how many tweets there are to process.

<h2 id="contraints">Constraints.</h2>
 * API Quota: We must request information form Twitter. We can run out of API quota per credential.
 * We can request just a 100 tweets at a time.
 * Each request has to go with the maximum amount of tweet ids (100) or the closest we can get to that, this to make the best use of our credentials.
 * Avoid tweet starvation when buffering tweets to get the 100 per request: If there are not enough tweets to flush a buffer after S seconds, then just flush the current amount.
 * Finite amount of credentials.
 * Tweets can come at any time, individually or in huge batches.
 * We need an easy to scale solution: A "just add more workers" approach must be enough.
 * Fail safe: If any worker suffers a sudden and horrible death it must not break the flow or lose tweets.
 * Update times for each tweet must follow its lifespan cycle (age) (younger - shortest interval).
 
 
<h2 id="background">Background.</h2>

<h3 id="references">Research references</h3>

<h4 id="tweet-lifespan">On a Tweet's lifespan</h4>

In the aricle [When Is My Tweet's Prime of Life? (A brief statistical interlude)](https://moz.com/blog/when-is-my-tweets-prime-of-life) by Peter Bay for [Moz](https://moz.com/), we can see a lot of statistics of a Tweets life span. He determines that 18 minutes are the average lifespan of a single tweet.

Another article called [The Short Lifespan of a Tweet: Retweets Only Happen Within the First Hour](http://readwrite.com/2010/09/29/the_short_lifespan_of_a_tweet_retweets_only_happen) by Frederic Lardinois for [readwrite](http://readwrite.com/) says that retweets (which some consider the currency of twitter) happen mostly during the first hour after its publication.

<h4 id="pipeline">On Computing Pipelines</h4>

Pipelines are workflows that can be found in many industries. The Production lines used in factories are pipelines where there is a series of steps in the process and each step is always working, when one step is done it provides input for the next step.

(1) In computing we are talking about data processing units in series in which one step's output serves as input for the next step.

[1.] <http://en.wikipedia.org/wiki/Pipeline_%28computing%29>

<h4 id="queueing">On Queueing theory</h4>

(1) Queueing theory refers to the study of waiting lines, in computing we use queues for jobs where processes that act as workers take jobs or receive jobs from the queues, they do their thing and notify the queue that their job is done, then they take another one or the queue sends them a new one.

[1.] <http://en.wikipedia.org/wiki/Queueing_theory>

<h3 id="dead-approaches">Dead Approaches</h3>

<h4 id="master-process-approach">Master Process Approach</h4>

This was the first idea proposed to solve this problem.
<a href="http://imgur.com/7xeXVoH"><img src="http://i.imgur.com/7xeXVoH.png" title="source: imgur.com" /></a>

This approach has a few problems.

* Master Process does way to many things.
* Scaling this approach requires the increase of Master Process instances and Updater Process instances.
* Requires locking and is defficient, just one lock is implemented when at least two should be implemented: one for preventing other Master process instances to access the same tweets about to be queued and the currenlty implemented lock when a tweet is being processed to prevent Master process instances from finding it again.
* Lock release logic is weak, if the Updater suffers a sudden and horrible death it might not release a tweet(s) causing deadlock and starving the locked tweet(s), this forces a bigger query to detect such starved tweets and enqueue them regardless their locked status (unreliable).
* There are way to many queries, one every Master Process cycle and they pull lots of data.
* Map reduce is responsible to get all tweets that will get updated, split them in groups of at most 100 elements and send them to queue, its logic is big and hard to mantain.

<h4 id="master-with-pipeline">The Master and Pipeline Approach</h4>

Another idea to fix the problem in a simpler way is to keep the Master Process pulling tweets, but this time instead of having this process finding the same tweets and re-scheduling them for update, it would just find those that were not processed ever, those tweets are sent to the Updater Pipeline where they will complete their processing. It relies completely on the pipeline to finish the jobs and the fetcher does a lot less stuff.

<a href="http://imgur.com/onmyknN"><img src="http://i.imgur.com/onmyknN.png" title="source: imgur.com" /></a>

* This approach removes some responsibility from Master Process.

But:

* Still needs two locks.
* The map reduce is still complex and big.
* Master Process decides at which point of the pipeline a tweet should go.

There must be a better simpler way.

<h3 id="last-idea">Keep just the pipeline</h3>


The Master Process is an unnecessary entity, we can send that guy with all his queries and locking complexity straight to oblivion.

We know that we are getting the tweets from some source, we don't really care which one. What we want to do is make that source start the pipeline by buffering the tweets and sending them to the job queue with a delay that matches the first interval of update.

The job queue will handle the entire flow and each node of the pipeline will re-use the queue to push the data to the next step.

<h2 id="hypothesis">Hypothesis.</h2>

**If:** We have a pipeline for incoming tweets in which each node updates the tweets in the correct interval (I<sub>n</sub>) passing the tweets to the next node through a delayed job queue.

**Then:** We will have updated tweets at any point of their lifespan while the job queue guarantees that no tweet gets lost in transit.

<h2 id="assumptions">Assumptions.</h2>
* A Job Queue capable of scheduling jobs (delayed jobs).
* On consistency and reliability: The job queue will not release jobs if they are not marked as finished, failed jobs will have retries.
* The job queue can work only in memory but if it does, it must keep a dump backup to recover the jobs if it crashes.
* On batches: A job includes just one tweet to request, the updater will build its buffer and flush when necessary.
* On intervals: They are provided by configuration in the shape of delays that can be applied in the job queue, they are not part of the problem to solve but given as input. Research ([Background on a tweet lifespan](#tweet-lifespan)) and the needs of the consumer platform produce these heuristics.
* The same queue is re used at the end of each node, delays to use will be selected by configuration given a tweet's age.
* Pipeline nodes are instances an Updater process (worker).
* Tweets and Retweets are treated the same by the updater.
* Tweets are provided to the system by a "starter" process, we consider them as our input.


<h2 id="solution">Solution</h2>

<h3 id="blackbox">Blackbox</h3>

#### The pipeline flow.
Looking at it as a flow, we would see many processes interacting with each other through a job queue.
<a href="http://imgur.com/4kPjtuH"><img src="http://i.imgur.com/4kPjtuH.png" title="source: imgur.com" /></a>

#### Looking closer to the queue and processes.
Removing one layer of abstraction we can see exactly how the Twitter Updater process interacts with a Delayed Queue to move one step ahead in the pipeline, thus making clear that the amount of actors is fairly small.

<a href="http://imgur.com/CCWSYw8"><img src="http://i.imgur.com/CCWSYw8.png" title="source: imgur.com" /></a>

<h3 id="toop">Theory of Operation</h3>

1. This process starts right after a tweet or retweet gets inserted into our system.
2. The Starter Process (SP) gets a tweet from an API call, a stream connection or any other input source.
3. SP does whatever it needs to do with the received tweet, save to the database might be one of the steps.
4. SP starts the pipeline by enqueueing the tweet with a delay of I<sub>1</sub> this delay is given by configuration.
5. Updater Process (UP) gets the job to update and pushes it into its buffer, if the buffer is empty at this point it will start a timer.
6. When the buffer is full or the timer reached its deadline, UP sends a request to 'statuses/lookup' with the buffered tweet ids and flushes the buffer.
7. Twitter will respond with the new information of the tweets.
8. UP should parse and save each tweet individualy.
9. For each tweet updated, if it should be re-scheduled the UP will grab the delay for the tweet and re-schedule it. If the tweet's update cycle is over ignore this step.


<h2 id="tech"> Technical Specification </h2>

### Job Queue with Scheduling/Delay capabilities.

It is assumed to exist, to be able to use it in the spec we will give it an API in the domain of the spec.

#### JobQue.schedule(namespace, data, delay)
Adds a job to the queue after the delay sent in milliseconds.

#### JobQueue.get(namespace)
Gets a job from a namespace.

**Input**:

* __namespace__ [String]: Namespace or category of the job, this string will be used to pull request jobs.
* __data__ [String]: JSON String with the data that will be used as input the worker processing this job.
* __delay__ [Number]: Time to wait in seconds before sending this job to the queue.

### Configuration ADT.

Configuration is assumed to be stored in key value pairs.
We suggest creating a wrapper (an Abstract Data Type) with set of functions to get such configuration and allow logic in it while hiding the actual data structure that holds the configuration.

#### Config.getTweetUpdateDelay(birthdate).

Given the age of a tweet this function determines how long to wait before the next update for this tweet.

**Data**:

```
  conf->tweetAge[18m]
  conf->tweetAge[60m]
  conf->tweetAge[1d]
  conf->tweetAge[7d]
  conf->tweetAge[30d]
```

Each key in tweetAge holds the interval of update when a tweet is younger than the time specified by the key.

**Input**:

 * __birthdate__ [Number]: The time in milliseconds that a tweet has lived.
 
**Output**:

 * __delay__ [Number]: The time in milliseconds to wait before updating this tweet. If -1 is returned it means you should not update this tweet.
 
** Algorithm **:

1. Let __conf__ be a configuration structure with access to a tweetAge structure with key value pairs.
2. Let __birthdate__ be the time in milliseconds that the tweet has lived.
3. If __birthdate__ is not provided return -1.
3. Let __now__ be the current timestamp.
4. Let __delta__ be the difference `now - birthdate`.
5. If __delta__ is between 0 and 1080000 (18 minues) return the value held by key '18m'.
6. If __delta__ is between 1080000 and 3600000 return the value held by key '60m'.
7. If __delta__ is between 3600000 and 86400000 return the value held by key '1d'.
8. If __delta__ is between 86400000 and 604800000 return the value held by key '7d'.
9. If __delta__ is between 604800000 and 2592000000 return the value held by key '30d'.
10. If __delta__ does not fit in any range return -1.


### Starter Process

A grey box.
It gets input from twitter, can be from a stream, from requests or any other input. You decide what sideffects you want here, like saving the tweet to db, parsing it, get stats from it, etc.
There is a part of this process that we need to control, at the end we will add a call to start the pipeline by queueing the tweet.

#### starterProcessInstance.queueTweet(tweet)

Given a tweet it enqueues it for the next update with the correct delay.

**Input**:

* __tweet__ [Object]: Instance of tweet.

**Algorithm**:

1. Let __t__ be an instance of a tweet.
2. __t__ must have an attribute id that holds the value of the `id_str` attribtue that Twitter sends with each tweet.
3. __t__ must have an attribute createdAt taht holds the value of the `created_at` attribute that Twitter sends with each tweet.
4. Let __delay__ be the waiting time before updating the tweet given by the call `Config.getTweetUpdateDelay(t.createdAt)`
5. Call `JobQueue.schedule()` with the following arguments `'tweet:update', {'data' : tweet.id}, delay`.


### Updater Process.

This worker runs continually, it never stops unless it fails, given that case the exception should be catched and ended gracefully.
The worker handles its internal state and most of its funcitons rely on such state.

#### Constants from configuration

* __bufferTTL__ [Number]: Time in milliseconds to wait before flushing a buffer/requesting buffered tweets.

#### State variables.

* __buffer__ [Array]: Holds the list of tweets that are going to be requestes, its maximum size is 100, once it reaches 100 elements it must be flushed.
* __startTime__ [Date]: Holds the date time when the buffer was empty but a first element or batch of elements gets inserted.



### run()

Starts the process. Recursive funciton, everytime it ends its flow it will call itself to start again. No arguments.

**Algorithm**:

1. Call `shouldStartTimer()`.
2. If response of `shouldStartTimer` is true then call `startTimer()`
3. Let __t__ be the result from calling `getTweetToUpdate()`.
4. If __t__ was returned and is not `null` then get it into the buffer by calling `appendToTweets(t)`
5. Call `shouldUpdate()`
6. If response of `shouldUpdate`is false then call self again `run()`
7. If response of `shouldUpdate`is true then call `requestTweets()`
8. Let __tweets__ be the response from calling `requestTweets()`
9. Call `saveTweets(tweets)`.
10. Call `promoteTweets()`. 
11. Call `emptyTweets()` to flush the __buffer__.
12. Call self again `run()`.

### shouldStartTimer()

Reads the tweets buffer and determines if the __startTime__ should be set.

**Algorithm**:

1. If buffer length is 0 return true.
2. Else return 1.


### startTimer()

**Algorithm**:

Set __startTime__ to current date.

### getTweetToUpdate()

Grabs a tweet from the Job queue.

**Algorithm**:

1. Call `JobQueue.get()`
2. Let __data__ be the job data from the previous call.
3. Get the __id__ from __data__.
4. Return __id__.

### appendToTweets(t)

Adds a tweet to the __buffer__.

**Algorithm**:

1. Get the __t__ tweet id.
2. Push __id__ to __buffer__.

### shouldUpdate(t)

Validates if the __buffer__ is full or if the __bufferTTL__ is depleated.

**Algorithm**:

1. If __buffer__ length is greater or equal to 100.
2. Return true
3. Let __delta__ be the difference betweeen the current date
4. If __buffer__ length is greater than 0 and __delta__ is greater or equal than __bufferTTL__.
5. Return true
6. If no condition was met return false.

### requestTweets()

Grabs ids from the __buffer__ and uses them to request information from using the Twitter's `statuses/lookup` REST API path.

**Algorithm**:

1. Let __query__ be the query parameter for the request.
2. Add an attribute `id` to __query__ using the joined values from __buffer__ separated by commas.
3. Send request to `statuses/lookup` with the __query__ parameter.
4. Get response.
5. Return response.


### saveTweets(tweets)

Gets the response from Twitter as input and stores them in your system.

**Algorithm**:

1. Let __tweets__ be an array of raw tweet objects, as Twitter sends them.
2. For each tweet.
3. Parse tweet, parsing algorithm depends on the information you need and the particular logic of your application.
4. Find your local record by using the tweet id.
4. Save parsed tweet to your DB.
5. After all tweets have return tweets.

### promoteTweets(tweets)

Sends the already updated tweets to the next step of the pipeline.

**Algorithm**:

1. Let __tweets__ be a collection of raw tweet objects.
2. For each tweet.
3. Call `Config.getTweetUpdateDelay(tweet.created_at)`.
4. Let __delay__ be the result from the previous call.
5. Call `JobQueue.schedule()` with the following arguments `'tweet:update', {'data' : tweet.id}, delay`.

### emptyTweets

Flushes the tweets buffer.

**Algorithm**:

1. __buffer__ = new Array.

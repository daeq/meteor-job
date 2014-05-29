meteor-job
======================================

**WARNING** This Package remains under development and the methods described here may change. As of now, there are no unit tests. You have been warned!

## Intro

Meteor Job is a pure Javascript implementation of the `Job` and `JobQueue` classes that form the foundation of the `jobCollection` Atmosphere package for Meteor. This package is used internally by `jobCollection` but you should also use it for any job workers you need to create and run outside of the Meteor environment as pure node.js programs.

Here's a very basic example that ignores authentication and connection error handling:

```js
var DDP = require('ddp');
var Job = require('meteor-job')

// In this case a local Meteor instance, could be anywhere...
var ddp = new DDP({
  host: "127.0.0.1",
  port: 3000,
  use_ejson: true
});

// Job uses DDP Method calls to communicate with the Meteor jobCollection.
// Within Meteor, it can make those calls directly, but outside of Meteor
// you need to hook it up with a working DDP connection it can use.
Job.setDDP(ddp);

// Once we have a valid connection, we're in business
ddp.connect(function (err) {
  if (err) throw err;

  // Worker function for jobs of type 'somejob'
  somejobWorker = function (job, cb) {
    job.log("Some message");
    // Work on job...
    job.progress(50, 100);  // Half done!
    // Work some more...
    if (jobError) {
      job.fail("Some error happened...");
    } else {
      job.done();
    }
    cb(null); // Don't forget!
  };

  // Get jobs of type 'somejob' available in the 'jobPile' jobCollection for somejobWorker
  workers = Job.processJobs('jobPile', 'somejob', somejobWorker);
});
```

## Installation

`npm install meteor-job`

Someday soon there will be tests...

## Usage

### Getting connected

First you need to establish a [DDP connection](https://github.com/oortcloud/node-ddp-client) with the Meteor server hosting the jobCollection you wish to work on.

```js
var DDP = require('ddp');
var Job = require('meteor-job')

// See DDP package docs for options here...
var ddp = new DDP({
  host: "127.0.0.1",
  port: 3000,
  use_ejson: true
});

Job.setDDP(ddp);

ddp.connect(function (err) {
  if (err) throw err;

  // You will probably need to authenticate here unless the Meteor
  // server is wide open for unauthenticated DDP Method calls, which
  // it really shouldn't be.
  // See DDP package for information about how to use:

    // ddp.loginWithToken(...)
    // ddp.loginWithEmail(...)
    // ddp.loginWithUsername(...)

  // The result of successfully authenticating will be a valid Meteor authToken.
  ddp.loginWithEmail('user@server.com', 'notverysecretpassword', function (err, response) {
    if (err) throw err;
    authToken = response.token

    // From here we can get to work, as long as the DDP connection is good.
    // See the DDP package for details on DDP auto_reconnect, and handling socket events.

    // Do stuff!!!

  });
}
```

### Job workers

Okay, so you've got an authenticated DDP connection, and you'd like to get to work, now what?

```js
// 'jobQueue' is the name of the jobCollection on the server
// 'jobType' is the name of the kind of job you'd like to work on
Job.getWork('jobQueue', 'jobType', function (err, job) {
  if (job) {
     // You got a job!!!  Better work on it!
     // At this point the jobCollection has changed the job status to 'running'
     // so you are now responsible to eventually call either job.done() or job.fail()
  }
});
```

However, `Job.getWork()` is kind of low-level. It only makes one request for a job. What you probably really want is to get some work whenever it becomes available and you aren't too busy:

```js
workers = Job.processJobs('jobQueue', 'jobType', { concurrency: 4 }, function (job, cb) {
  // This will only be called if a job is obtained from Job.getWork()
  // Up to four of these worker functions can be oustanding at
  // a time based on the concurrency option...

  cb(); // Be sure to invoke the callback when this job has been completed or failed.

});
```

Once you have a job, you can work on it, log messages, indicate progress and either succeed or fail.

```js
// This code assumed to be running in a Job.processJobs() callback
var count = 0;
var total = job.data.emailsToSend.length;
var retryLater = [];

// Most job methods have optional callbacks if you really want to be sure...
job.log("Attempting to send " + total + " emails", function(err, result) {
  // err would be a DDP or server error
  // If no error, the result will indicate what happened in jobCollection
});

job.progress(count, total);

if (networkDown()) {
  // You can add a string message to a failing job
  job.fail("Network is down!!!");
  cb();
} else {
  job.data.emailsToSend.forEach(function (email) {
    sendEmail(email.address, email.subject, email.message, function(err) {
      count++;
      job.progress(count, total);
      if (err) {
        job.log("Sending email to " + email.address + "failed"., {level: 'warning'});
        retryLater.push(email);
      }
      if (count === total) {
        // You can attach a result object to a successful job
        job.done({ retry: retryLater });
        cb();
      }
    });
  });
}
```

However, the retry mechanism in the above code seems pretty clunky... How do those failed messages get retried?
This approach probably will probably be easier to manage:

```js
workers = Job.processJobs('jobQueue', 'jobType', { payload: 20 }, function (jobs, cb) {
  // jobs is an array of jobs, between 1 and 20 long, triggered by the option payload > 1
  var count = 0;

  jobs.forEach(function (job) {
    email = job.data.email // Only one email per job
    sendEmail(email.address, email.subject, email.message, function(err) {
      count++;
      if (err) {
        job.log("Sending failed with error" + err, {level: 'warning'});
        job.fail("" + err);
      } else {
        job.done();
      }
      if (count === jobs.length) {
        cb();  // Tells the processJobs we're done
      }
    });
  });
});
```

With the above logic, each email can succeed or fail individually, and retrying later can be directly handled by the jobCollection itself.

The jobQueue object returned by `Job.processJobs()` has methods that can be used to determine its status and control it's behavior. See the jobQueue API reference for more detail.

### Job creators

If you'd like to create an entirely new job and submit it to a jobCollection, here's how:

```js
job = new Job('jobQueue', 'jobType', { work: "to", be: "done" });

// Set some options on the new job before submitting it. These option setting
// methods do not take callbacks because they only affect the local job object.
// See also: job.repeat(), job.after(), job.depends()

job.priority('normal')                    // These methods return job and so are chainable.
   .retry({retries: 5, wait: 15*60*1000}) // Retry up to five times, waiting 15 minutes per attempt
   .delay(15000);                         // Don't run until 15 seconds have passed

job.save(function (err, result) { //Save the job to be added to the Meteor jobCollection via DDP
  if (!err && result) {
    console.log("New job saved with Id: " + result);
  }
});
```

**Note:** It's likely that you'll want to think carefully about whether node.js programs should be allowed to create and manage jobs. Meteor jobCollection provides an extremely flexible mechanism to allow or deny specific actions that are attempted outside of trusted server code. As such, the code above (specifically the `job.save()`) may be rejected by the Meteor server depending on how it is configured. The same caveat applies to all of the job management methods described below.

### Job managers

Management of the jobCollection itself is accomplished using a mixture of Job class methods and methods on individual job objects:

```js
// Get a job object by Id
Job.getJob('jobQueue', id, function (err, job) {
  // Note, this is NOT the same a Job.getWork()
  // This call returns a job object, but does not change the status to 'running'.
  // So you can't work on this job.
});

// If your job object's infomation gets stale, you can refresh it
job.refresh(function (err, result) {
  // job is refreshed
});

// Make a job object from a job document (which you can obtain by subscribing to a jobCollection)
job = Job.makeJob('jobQueue', jobDoc);  // No callback!
// Note that jobCollections are reactive, just like any other Meteor collection. So if you are
// subscribed, the job documents in the collection will autoupdate. Then you can use Job.makeJob
// to turn a job doc into a job object whenever necessary without another DDP roundtrip

// Once you have a job object you can change many of its settings (but only while it's paused)
job.pause(function (err, result) {   // Prohibit the job from running on the queue
  job.priority('low');   // Change its priority
  job.save();            // Update its priority in the jobCollection
                         // This also automatically triggers a job.resume()
                         // which is how you'd otherwise get it running again.
});

// You can also cancel jobs that are running or are waiting to run.
job.cancel();

// You can restart a cancelled or failed job
job.restart();

// Or re-run a job that has already completed successfully
job.rerun();

// And you can remove a job, so long as it's cancelled, completed or failed
// If its running or in any other state, you'll need to cancel it before you can remove it
job.remove();

// For bulk operations on acting on more than one job at a time, there are also Class methods
// that take arrays of job Ids.  For example, cancelling a whole batch of jobs at once:
Job.cancelJobs('jobQueue', Ids, function(err, result) {
  // Operation complete. result is true if any jobs were cancelled (assuming no error)
});
```

## API

### class Job

`Job` has a bunch of Class methods and properties to help with creating and managing Jobs and getting work for them.

#### `Job.setDDP(ddp)`

This class method binds `Job` to a specific instance of `DDPClient`. See [node-ddp-client](https://github.com/oortcloud/node-ddp-client) for more details. Currently it's only possible to use a single DDP connection at a time.

```js
var ddp = new DDP({
  host: "127.0.0.1",
  port: 3000,
  use_ejson: true
});

Job.setDDP(ddp);
```

#### `Job.getWork(root, type, [options], [callback])`

Get one or more jobs from the job Collection, setting status to `'running'`.

`options`:
* `maxJobs` -- Maximum number of jobs to get. Default `1`  If `maxJobs > 1` the result will be an array of job objects, otherwise it is a single job object, or `undefined` if no jobs were available

`callback(error, result)` -- Optional only on Meteor Server with Fibers. Result will be an array or single value depending on `options.maxJobs`.

```js
if (Meteor.isServer) {
  job = Job.getWork(  // Job will be undefined or contain a Job object
    'jobQueue',  // name of job Collection
    'jobType',   // type of job to request
    {
      maxJobs: 1 // Default, only get one job, returned as a single object
    }
  );
} else {
  Job.getWork(
    'jobQueue',                 // root name of job Collection
    [ 'jobType1', 'jobType2' ]  // can request multiple types in array
    {
      maxJobs: 5 // If maxJobs > 1, result is an array of jobs
    },
    function (err, jobs) {
      // jobs contains between 0 and maxJobs jobs, depending on availability
      // job type is available as
      if (job[0].type === 'jobType1') {
        // Work on jobType1...
      } else if (job[0].type === 'jobType2') {
        // Work on jobType2...
      } else {
        // Sadness
      }
    }
  );
}
```

#### `Job.processJobs(root, type, [options], worker)`

Create a `JobQueue` to automatically get work from the job Collection, and asyncronously call the worker function.

See the `JobQueue` section for documentation about the methods and attributes on a `JobQueue` instance.

`options:`
* `concurrency` -- Maximum number of async calls to `worker` that can be outstanding at a time. Default: `1`
* `cargo` -- Maximum number of job objects to provide to each worker, Default: `1` If `cargo > 1` the first paramter to `worker` will be an array of job objects rather than a single job object.
* `pollInterval` -- How often to ask the remote job Collection for more work, in ms. Default: `5000` (5 seconds)
* `prefetch` -- How many extra jobs to request beyond the capacity of all workers (`concurrency * cargo`) to compensate for latency getting more work.

`worker(result, callback)`
* `result` -- either a single job object or an array of job objects depending on `options.cargo`.
* `callback` -- must be eventually called exactly once when `job.done()` or `job.fail()` has been called on all jobs in result.

```js
queue = Job.processJobs(
  'jobQueue',   // name of job Collection
  'jobType',    // type of job to request, can also be an array of job types
  {
    concurrency: 4,
    cargo: 1,
    pollInterval: 5000,
    prefetch: 1
  },
  function (job, callback) {
    // Only called when there is a valid job
    job.done();
    callback();
  }
);

// The job queue has methods... See JobQueue documentation for details.
queue.pause();
queue.resume();
queue.shutdown();
```

#### `Job.makeJob()`

#### `Job.getJob()`

#### `Job.startJobs()`

#### `Job.stopJobs()`

#### `Job.getJobs()`

#### `Job.pauseJobs()`

#### `Job.resumeJobs()`

#### `Job.cancelJobs()`

#### `Job.restartJobs()`

#### `Job.removeJobs()`

The following Job class attributes define various states and levels used by `jobCollection`

#### `Job.forever`

#### `Job.jobPriorities`

#### `Job.jobStatuses`

#### `Job.jobLogLevels`

#### `Job.jobStatusCancellable`

#### `Job.jobStatusPausable`

#### `Job.jobStatusRemovable`

#### `Job.jobStatusRestartablee`

Objects that are instances of Job

#### `j = new Job()`

#### `j.depends()`

#### `j.priority()`

#### `j.retry()`

#### `j.repeat()`

#### `j.delay()`

#### `j.after()`

#### `j.log()`

#### `j.progress()`

#### `j.save()`

#### `j.refresh()`

#### `j.done()`

#### `j.fail()`

#### `j.pause()`

#### `j.resume()`

#### `j.cancel()`

#### `j.restart()`

#### `j.rerun()`

#### `j.remove()`

#### `j.type`

#### `j.data`

### class JobQueue

JobQueue is similar in spirit to the [async.js](https://github.com/caolan/async) [priorityQueue](https://github.com/caolan/async#priorityQueue) except that it gets its work from the Meteor jobCollection via calls to `Job.getWork()`

#### `q = Job.processJobs()`

#### `q.resume()`

#### `q.pause()`

#### `q.shutdown()`

#### `q.length()`

#### `q.full()`

#### `q.running()`

#### `q.idle()`


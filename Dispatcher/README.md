#Dispatcher
The Dispatcher is responsible for receiving new Job requests and assigning those Jobs to Workers. The Dispatcher is also responsible for periodically checking on the health of both Jobs and Workers and maintaining a list of active Jobs and knowing what the capacity of the system is.

One major requirement that makes the system work smoothly is that all jobs must be re-runnable and abortable. If a Worker dies, the job will automatically be reassigned to another Worker. There should be no damage done if a Job is only half complete and then re-started from scratch.

Using upserts instead of inserts and using temp collections (that include the run ID of the Workflow) that will be dropped and re-built is the best way to not to create collisions if somehow the same job is run more than once.

TODO: In the event of a network partition, where the Worker can no longer communicate with the Dispatcher, after a certain number of retries, it should shutdown (self-destruct) to prevent the Job it currently knows about from being executed twice at the same time.

#Persistent Storage and Failover
Both Jobs and Workers are stored in Mongo and shared by all the Dispatchers. Dispatchers are run in a Master-Master configuration and they all share the same queue of Jobs and the same registry of Workers. Dispatchers have no knowledge of each other and all of them will act independently.

Each of the actions that they take (pinging workers, assigning jobs, verifying jobs) are designed to be collision-safe (there is no harm if they are repeated, or they prevent duplicate actions) so the number of Dispatchers can be arbitrarily large.

#WorkerSchema

The worker schema defines what worker properties will be saved in a database, i.e. how a worker "appears" to the dispatcher.

Each worker provides a hostname (or IP address), port, and maximum number of jobs it can handle to its dispatcher on startup. These values are required by the WorkerSchema's constructor, and should be passed in when each new object is created.

this.Schema = mongoose.Schema({
  // The hostname/IP address of the worker. This is recorded by the dispatcher when the worker registers.
  hostname: {type: 'String', required: true},
  // The port to connect to. Like the hostname, this is recorded on startup.
  port: {type: Number, required: true},
  // The types of jobs this worker supports
  handlers: {},
  // The worker tells the dispatcher on startup what is the maximum amount of jobs it can hold.
  maxJobs: {type: Number, required: true},
  // A list of all the jobs the worker is currently handling.
  jobs: [String],
  // When was the last time we pinged this worker?
  lastPinged: {type: Date, 'default':Date.now, required: true},
  // Indicates how many tries since the last successful ping
  failedPingCount: {type: Number, 'default':0, required:true},
  // The last error received when pinging this worker
  lastError: {},
  // Did the worker resign?
  isResigned: {type: Boolean, 'default':false, required:true},
  // Is the worker unreachable/unhealthy?
  isDead: {type: Boolean, 'default':false, required:true}
});
#Job Schema

The Dispatcher uses the same Job Schema as defined in the Main README.

#Dispatching Jobs
The primary role of the Dispatcher is to intelligently assign Jobs to the Workers with the most capacity available.

This is the one process that should not be repeated by multiple Dispatchers. To prevent the same Job from being Dispatched multiple times simultaneously there is a Job Cooldown Period, specified by the dispatcher.config.jobs.cooldown_period parameter (in milliseconds).

The process for dispatching a Job is: 1. Dispatcher finds the first Job in the queue 2. Saves the Job with a status of d for dispatching and puts its own ID* in the dispatched_by property 3. Dispatcher waits for the cooldown period 4. Dispatcher re-reads the Job and checks if the ID specified is still its own 5. a) If it is, then dispatch the job b) if not, then do nothing

*ID - Dispatcher IDs are generated upon startup and are created through a combination of the random number function and the timestamp, making it nearly impossible to have two Dispatchers with the same ID. They would have to be started at the exact same millisecond and even then it would be a 1/100,000 chance they would get the same ID.

The cooldown period makes certain that all Dispatchers who were attempting to dispatch the Job simultaneously will have a chance to attempt to save the Job as d with their own ID. After that happens, then it is just the Dispatcher who saved last who wins the right to dispatch the Job.

TODO: If the designated Dispatcher dies before it gets a chance to dispatch the job all dispatching will come to a halt. We need to add some sort of check to make sure that jobs that have been waiting to be dispatched for too long are put back into the queue.

#Checking the Status of Jobs
Once a Job has been dispatched, there are several checks that need to be made to ensure the stability of the system:

The Worker document specified by the execution.worker_id property on the Job contains the Job in its array of active Jobs
The Worker server knows about the Job (check via HTTP request) and is currently processing it
All of the Jobs listed as in-progress in each of the Worker Documents match valid in-progress jobs in the job collection
Any Job that has reached its timeout as specified by the execution.expiry property, is marked as f or failed status and a cancellation is sent to the Worker.
Managing Workers
The Dispatcher maintains an active registry of Workers with a list of how many simultaneous Jobs they can handle, what types of Jobs they can handle, and what their hostname and port is.

#Registering Workers

Workers on start-up will make a PUT request to /dispatcher/workers/ to create an entry for itself in the Dispatcher's database. Once the worker is registered the Dispatcher will begin to periodically check on the health of that Worker.

#Checking Workers

The Dispatcher will check on the health of a Worker by making a request to its /health_check/ route based on the parameter in the dispatcher.config called workers.ping_interval. This should be set to something reasonable so we aren't spamming our workers with health check requests.

#Resigning Workers

When a worker notifies the Dispatcher that it would like to resign (a DELETE request) the Dispatcher checks if any active Jobs are assigned to that Worker, and if there are Jobs, it will re-assign them back to the queue.

Antfarm
A batch processing library used as the aggregation engine for the Home Page Reports and for the batch process execution engine for Scheduled Reports (future).



![My Screenshot](http://newuser02.github.com/newuser02/images/Test1.png)




It is designed to be a highly-available cluster to perform any job that it knows how to perform.

Table of Contents
Terms
Components
System Architecture
Job Structure
Errors
Abstract Classes
The Service Container
Databases
Configuring the App
Running the App
Running the Tests
Terms
Some terms to know before we get started:

Job - A unit of work to be done, like making a request to a URL and dumping the contents into mongo, or sending an email, or updating some DB records, etc.
Job Run - A specific instance of a job that is currently being or has been executed
Workflow - A set of jobs that should be performed together
Workflow Configuration - The configuration object that determines when a workflow should begin and the various settings that should be used to run it
Workflow Run - A specific run of a Workflow
Trigger - A set of conditions to be met for a Workflow to begin
Components
The Antfarm project is made up of 4 main components:

Workflow Manager
Dispatcher
Worker
Logger
The Workflow Manager is responsible for managing when Workflows begin and end. The trigger for starting a Workflow can be time based (a schedule) or can be based on any number of external events (especially other jobs). When the criteria to begin a Workflow is met, the Workflow Manager will notify the Dispatcher that a new Job should be created. The Job created will be the first Job in the Workflow.

The Dispatcher will receive requests to create new Jobs, and will distribute the Jobs it receives evenly to the many Workers that it knows about. The Dispatcher is responsible for managing the active Jobs and for maintaining a queue of jobs to be completed. The Dispatcher also manages a history of completed jobs and responds to inquiries regarding the status of any existing job. There can be many Dispatchers in the system and they all share the same persisted queue storage for communication.

Workers receive jobs to perform from a Dispatcher. The Job will contain information about the task to be performed, such as the URL for a Job that requires an HTTP request, or the name of the file to save as, for a Job that requires a file to be generated. When a Job is completed, the Worker will respond to the Dispatcher that requested the Job indicating that it is complete and the results requested (if any) for that Job.

Upon receiving confirmation about the completion of a Job, the Dispatcher will make a callback request (if one is specified in the Job) to the requestor, to indicate that the Job completed, and the results of completion. In the case of the Workflow Manager, the response will trigger the next step in the Workflow.

System Architecture
The Workflow Manager, Dispatcher, and Worker applications will each live on separate servers. For redundancy, there will be many of each type of server.

The relationships between each of these systems are described below.

Relationships

Between the Workflow Manager and the Dispatcher:

A Workflow Manager will work with A Dispatcher
In practice, the Dispatchers will be load-balanced behind a common domain name because their as long as the Dispatchers are working in a cluster, it doesn't matter which one you make the request to.

Workflow Managers will work in a Master-Slave configuration with the slaves maintaining communication to ensure the health of the master. The Master-Slave configuration will prevent duplicate Job creations for the same Job. Only the Master will request jobs to be created from the Dispatcher and will be notified when Jobs have completed.

Between the Dispatcher and the Worker:

A Dispatcher will have Many Workers
Between the Worker and the Dispatchers

A Worker will have Many Dispatchers
Job Structure
All of the components share a Job Structure which will be passed around as JSON through various requests. Take a look at the general job structure below:

this.Schema = mongoose.Schema({
  workflow: String,                                  // The name of the workflow this job belongs to (if any)
  run: Number,                                       // The run of the workflow this job belongs to (if any)
  created: {type: Date, 'default': Date.now},        // When the job was created - tagged by the dispatcher
  priority: {type: Number, 'default':0},             // To be used in the future for sequencing jobs
  status: {type: String, enum: self.valid_statuses}, // An enumerated single char status
  result: {},                                        // An object containing the result of the job run
  updated: {type: Date, 'default': Date.now},
  dispatcher: {
    id: {type: Number, 'default':0},                 // The ID of the dispatcher that will be assigning this job
    expiry: Date,                                    // The time when the job should be dispatched,
    date: {type: Date}
  },
  execution: {
    worker_id: String,                               // The ID of the Worker executing the Job
    started: { type: Date },
    ended: { type: Date },
    expiry:  { type: Date },                         // Time when the job will be killed
    timeout: {type:Number, 'default':3600 * 1000}    // Milliseconds before a job is canceled and marked as failed due to timeout
  },
  type: String,                                      // The job to be performed
  payload: {},                                       // Job-specific info
  callback: {                                        // Callback to post the completed Job information to
    hostname: String,
    port: Number,
    path: String,
    method: {type:String, enum: ['GET', 'PUT', 'POST', 'DELETE']},
    headers: {},
    didCallbackSuccessfully: {type: Boolean, 'default':false} // Were we able to callback after the job finished?
  }
});
Errors
Error handling is hugely important in an application of this size. In an attempt to make it simple to identify and track errors, we have standardized the error response format and have created an index of possible errors throughout the app.

Structure

Throughout the application there are various HTTP endpoints that may return a 400 response. In order to standardize the errors returned, the output error should be in the job and will be in the result property. The Job Status will be FAILED and the job result will have whatever the error actually was.

Possible Errors

A listing of potential errors that can be seen throughout the application in regards to Workflow and Job execution.

Workflow Manager

TODO

Dispatcher

TODO

Worker

TODO

Abstract Classes
Most of the components in this project extend from an Abstract version of themselves. This helps increase the reusability by forcing us to think about and implement a common API for all of the elements that we use. For example, in the Storage classes that are used in the Worker Job Handlers, we have a Mongo type, a GridFS type, and a Pipe type, which all adapt the underlying data storage clients to our interface to make it simple to interchange storage engines without actually making code changes to the Handler itself.

The Service Container and Depency Injection
Throughout this project we made heavy use of a Symfony2-style Service Container to provide dependency injection throughout our classes. This provides us with the opportunity to abstract dependencies and vastly increases the reusability and testability of all of our classes. This is especially visible in the Worker Handlers which basically reuse the DataTransformationHandler many times with a different Transformer class to do different things.

We created a Node.js port of the Symfony2 Service Container and the code is freely available here.

Databases
TODO: NOT YET IMPLEMENTED

Each sub-component of the antfarm project can technically run completely independently of all of the other components, including the database that it utilizes. To emphasize this point, the databases are separated into:

antfarm_workflows_[ENV]
antfarm_jobs_[ENV]
antfarm_data_[ENV]
antfarm_logs_[ENV]
Where each serves a different sub-system.

The antfarm_data DB will be the largest because it will contain all of the results from the jobs that have been performed.

Configuring the Application
The various components and settings can all be configured through the parameters.json file in the root directory. The parameters.json file will override the default values specified in the various Service Container configuration files and will allow you to control DB connections, ports that the different apps listen on, and even what hosts they originate from. The parameters.json file lives outside of version control because the parameters contained within it should be related to a specific deployment of the application. The parameters_example.json file will show you what parameters are available, their default values, and contain a comments block to explain what the various parameters are and their settings. Below is a copy of the file with the comment block included.

{
  "comments" : {

    "README":"This is a fake block I added that does not do anything but have info and should be removed",

    "parameter_help": {
      "mongoose_connection.config":{
        "used_by":["Dispatcher", "Workflow Manager"],
        "purpose":"Provides persistent storage for workflows and job and worker metadata",
        "see":"http://mongodb.github.io/node-mongodb-native/api-generated/db.html",
        "defined_in":"antfarm/services.json"
      },
      "mongo_storage.config":{
        "used_by":["Worker"],
        "purpose":"Output from jobs are placed into this database (both GridFS and Mongo)",
        "see":"http://mongodb.github.io/node-mongodb-native/api-generated/db.html",
        "defined_in":"antfarm/Worker/storage.json"
      },
      "gridfs_storage": {
        "used_by":["Worker"],
        "purpose":"Output from HTTP Request and Encoding jobs go into GridFS as binary data - defines GridFS options, uses mongo_storage",
        "see":"http://mongodb.github.io/node-mongodb-native/api-generated/gridstore.html",
        "defined_in":"antfarm/Worker/storage.json"
      },
      "worker.config": {
        "used_by":["Worker"],
        "purpose":"Define the hostname or the network adapter and the max number of jobs for this worker",
        "see":"http://github.private.linksynergy.com/SD/antfarm/blob/dev/Worker/README.md",
        "defined_in":"antfarm/Worker/interfaces.json"
      },
      "worker_http_interface.config": {
        "used_by":["Worker"],
        "purpose":"Define the port for the Worker's HTTP interface and how to connect to the Dispatcher",
        "defined_in":"antfarm/Worker/interfaces.json"
      },
      "dispacher.config": {
        "used_by":["Dispatcher"],
        "purpose":"Defines the various check intervals for jobs and workers",
        "see":"http://github.private.linksynergy.com/SD/antfarm/blob/dev/Dispatcher/README.md",
        "defined_in":"antfarm/Dispatcher/services.json"
      },
      "dispatcher_http_interface.config": {
        "used_by":["Worker"],
        "purpose":"Define the port for the Dispatcher to listen on",
        "defined_in":"antfarm/Dispatcher/services.json"
      },
      "wm_http_interface.config": {
        "used_by":["Worker"],
        "purpose":"Define the port for the Workflow Manager's HTTP interface and how to connect to the Dispatcher",
        "defined_in":"antfarm/WorkflowManager/services.json"
      },
    }
  },

  "parameters": {
    "mongoose_connection.config": {
      "uri":"mongodb://localhost:27017/antfarm",
      "db_options": {
        "user":"antfarm",
        "pass":"antfarm",
        "server": {
          "socketOptions": { "keepAlive": 1 }
        },
        "replset": {
          "rs_name":"optimus_rs_1",
          "socketOptions": { "keepAlive": 1 }
        }
      }
    },
    "mongo_storage.config": {
      "w":1
    },
    "mongo_storage.user":"antfarm",
    "mongo_storage.pass":"antfarm",
    "mongo_storage.db_name":"antfarm",
    "mongo_storage.repl_set_servers": [
      {"host":"gcvs0172.private.linksynergy.com", "port":27017},
      {"host":"gcvs4100.private.linksyerngy.com", "port":27017}
    ],
    "gridfs_storage.write.config":{
      "mode":"w",
      "chunk_size":51200,
      "options": {
        "root":"fs"
      }
    },
    "gridfs_storage.read.config":{
      "mode":"r",
      "chunk_size":51200,
      "options": {
        "root":"fs"
      }
    },
    "worker.config": {
      "hostname": null,
      "max_jobs": 1,
      "should_use_hostname": false,
      "network_adapter": "eth1"
    },
    "worker_http_interface.config": {
      "port": 8082,
      "dispatcher": {
        "host": "localhost",
        "port": 8081
      }
    },
    "dispatcher.config": {
      "jobs": {
        "cooldown_period": 2000,
        "job_verification_interval": 10000
      },
      "workers": {
        "ping_interval": 30000,
        "max_retries": 3,
        "dead_worker_ping_interval": 60000,
        "dead_worker_max_retries": 10
      }
    },
    "dispatcher_http_interface.config": {
      "port": 8081
    },
    "wm_http_interface.config": {
      "port": 8083,
      "dispatcher": {
        "host": "localhost",
        "port": 8081
      }
    }
  }
}

In table format, the list of parameters are as follows:

Comp	Parameter	Type	Description
D/WM	mongoose_connection.config.uri	Str	MongoDB Connection URI for Mongoose
D/WM	mongoose_connection.config.db_options	Obj	See the Mongo Native Driver Options
W	mongo_storage.config	Obj	Options for Worker Mongo connection, 
both Mongo and GridFS. See the Mongo Native Driver Options
W	mongo_storage.user	Str	DB user for the Worker
W	mongo_storage.pass	Str	DB pass for the Worker
W	mongo_storage.db_name	Str	DB name for the Worker
W	mongo_storage.repl_set_servers	Arr	Array of objects specifying the Mongo 
replica set {"host":"gcvs0172.private.linksynergy.com", "port":27017}
W	gridfs_storage.write.config	Obj	Options for Worker GridFS write connection.
See the GridStore Options
W	gridfs_storage.read.config	Obj	Options for Worker GridFS read connection.
See the GridStore Options
W	worker.config.hostname	Str	A DNS name for this server, if none is provided,
the IP will be looked up based on the Network Adapter specified
W	worker.config.max_jobs	Num	The number of Jobs this Worker can execute 
simultaneously
W	worker.config.should_use_hostname	Bool	Use the hostname as specified by the OS in 
os.hostname() instead of the IP
W	worker.config.network_adapter	Str	The Network Adapter to use to get the 
Worker IP
W	worker_http_interface.config.port	Num	The port for the HTTP server to listen on
W	worker_http_interface.config.dispatcher	Obj	The hostname and port of the Dispatcher 
{"hostname":"dispatcher.linksynergy.com","port":8080}
D	dispatcher.config.jobs.cooldown_period	Num	The number of ms to wait before dispatching 
a Job, see the Dispatcher docs.
D	dispatcher.config.
jobs.job_verification_interval	Num	The number of ms between verifying 
in-progress Jobs and Jobs assigned to Workers, see the Dispatcher docs.
D	dispatcher.config.workers.ping_interval	Num	Number of ms between calls to the 
Worker's health check, see the Dispatcher docs.
D	dispatcher.config.workers.max_retries	Num	Number of failed health checks 
before a Worker is marked isDead, see the Dispatcher docs.
D	dispatcher.config.
workers.dead_worker_ping_interval	Num	Number of ms between pings to a 
dead Worker, slow down checks when a Worker is dead, see the Dispatcher docs.
D	dispatcher.config.
workers.dead_worker_max_retries	Num	Number of failed attempts before a 
Worker is removed completely, see the Dispatcher docs.
D	dispatcher_http_interface.config.port	Num	Port for the HTTP server to listen on
D	wm_http_interface.config.port	Num	Port for the HTTP server to listen on
D	wm_http_interface.config.dispatcher	Num	Hostname and port of the Dispatcher
Running the Application
The application can be run from bin/antfarm.js and can be run like:

node bin/antfarm.js (workflow_manager|worker|dispatcher)
The ports that they run on are hardcoded for now and the things that they do are also hardcoded for now.

You can start all 3 processes on the same machine for now and the Workflow Manager will try to load data into Mongo starting with Jan 1, 2013.

Because the data volume is too large for one day, it will filter each day by MID and page through MIDs. This is more efficient than row count based paging because there is a bitmap index on MID in oracle.

# Autoscale Triggers
An IronWorker that scales up other workers based on IronMQ queue sizes.  You're welcome to fork/extend this repo or create your own.

## Local Testing
If you do not have Go:
```shell
docker run --rm -it -v $PWD:/go/src/a -w /go/src/a -e "CONFIG_FILE=config.json" iron/go:dev sh -c "go get ./... && go run main.go"
```

If you have Go:
```shell
go get github.com/NickSardo/triggers
cd $GOPATH/src/github.com/NickSardo/triggers
export CONFIG_FILE=config.json
go run main.go
```

## Example configuration
```json
{
	"envs": {
		"q":{
    			"token": "AAA",
    			"project_id": "BBB",
                "host": "mq-aws-us-east-1-1.iron.io",
				"api_version": "3"
		},
		"w":{
    			"token": "AAA",
    			"project_id": "CCC"
		}
	},
	"alerts": [
			{
			"queueName": "sampleQueue",
			"queueEnv": "q",
			"workerName": "dequeuer",
			"workerEnv": "w",
			"triggers":[
				{
					"type":"ratio",
					"value": 10
				}
			],
			"cluster":"ABC"
		}
	],
	"cacheEnv":"w",
	"interval": 10,
	"runtime": 1800
}
```

`envs`: named map of bjects describing different Iron.io environments. See dev.iron.io for more information  
`alerts`: Array of objects, each object connects a queue to monitor and a worker to start   
`queueEnv` or `workerEnv`: these values are found under defined environments above  
`cluster`: tasks for this worker are spawned on this cluster   
`cacheEnv`: this scaler code caches the last known queue size, provide an environment to Iron Cache  
`interval`: polling frequency for checking the queue sizes & launching tasks (optional, default: 10 sec)    
`runtime`: seconds this task will live for (optional, default: 1800 seconds)  

#### Triggers
Triggers will tell the scaler how many tasks to spawn.  Given more than one trigger to an alert object, the max tasks generated among all triggers will be used.

##### `ratio` (recommended)
Have one task queued or running for every {value} messages on the queue.

Example:   
Given a ratio value of 10, `# of tasks = queue size / 10`
<img src="ratio.png" alt="Ratio Trigger" width="700" height="350">

##### `fixed`
If the queue size >= {value}, have at least one task queued or running.

Example:  
Given a fixed value of 50, have at least one task queued or running starting at time = 20 min.
<img src="fixed.png" alt="Fixed Trigger" width="700" height="350">

##### `progressive`
If the queue size grows by {value} messages since the last check time, spawn 1 task. Note this is affected by the `interval` parameter.  
Given a progressive value of 20 and a check interval of 1 minute, spawn two tasks.   
`# of tasks created = (current - prev) / 20`  
<img src="progressive.png" alt="Progessive Trigger" width="700" height="350">

## Deploying to IronWorker
If you want to skip compiling the code yourself, you can go to step 3 and use `nicksardo/triggers:0.1`

##### 1. Build this executable
```shell
docker run --rm -it -v $PWD:/go/src/a -w /go/src/a iron/go:dev sh -c "go get ./... && go build -o triggers"
```

##### 2. Build dockerfile and push to your docker registry
```shell
docker build -t {{youraccount}}/triggers:0.1 .
docker push {{youraccount}}/triggers:0.1
```

##### 3. Test again by creating a config file and running your docker image
```shell
vi prod_config.json  # create a config file called "prod_config.json" in an empty directory
docker run --rm -it -e "CONFIG_FILE=prod_config.json" -v $PWD:/app {{youraccount}}/triggers
```

##### 4. Register your docker image with Iron.io
```shell
# Upload
iron register {{youraccount}}/triggers:0.1
```

##### 5. Set configuration data
Go to HUD and copy/paste your prod_config.json contents to the worker configuration.


##### 6. Schedule task every 30 minutes
```
# Schedule this to run every 30 minutes via CLI or HUD
iron worker schedule -cluster default -run-every 1800 {{youraccount}}/triggers
```

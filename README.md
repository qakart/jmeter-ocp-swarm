# Jmeter on OCP

This project is a collection of samples file to run jmeter in remote/controller mode.

You can read the official jmeter [documentation](http://jmeter.apache.org/usermanual/remote-test.html) to run jemter in remote/controller mode.

# Create the demo app

First you need to build the app, then:

```oc new-build --image-stream=openshift/jboss-webserver31-tomcat8-openshift:1.1 --name=cool-app --binary=true```

You should see this from the web console:

Empty project

We can now start the binary deploy (build the cool-app with maven from the web-app dir):

```oc start-build cool-app --from-file=<PATH_TO>/ROOT.war```

Deploy a new app

```oc new-app cool-app```

Create the route:

```oc expose svc/cool-app```

# Create server slave image

Point to dir 

```[PATH-TO]/jmeter-ocp-swarm/jmeter-server-remote/jmeter-slave-image```

Create the new build:

```oc new-build --strategy docker --binary --name jmeter-server-remote```

Build our image:

```oc start-build jmeter-server-remote --from-dir . --follow```

Create at least one running pod:

``` oc new-app jmeter-server-remote```

The above command will start a new pod from the previously create build 

*NOTE*: all the instances are started with ```-Dserver.rmi.ssl.disable=true``` to turn off SSL.

# Create jmeter controller image

Go to dir 

```[PATH-TO]/jmeter-swarm/meter-controller/jmeter-controller-image```

Create the new build:

```oc new-build --strategy docker --binary  --name jmeter-controller```

Build our image:

```oc start-build jmeter-controller --from-dir . --follow```

## Setup the Jmeter project

In this example we use a config map to store the jmeter projet to run.
The config map is mounted by the Jmeter controller job.

Go to dir 

```[PATH-TO]/jmeter-swarm/meter-controller/demo```

```oc create configmap jmeter-test --from-file=cool-app-jmeter.jmx```

## Running jmeter controller as job

Create a job yaml file from the template, the configurables parameters are:

* TEST_PLAN_NAME the name of the jmeter file
* TEST_CONFIG_MAP_NAME the name of the config map where the jmeter test is stored
* REMOTE_SERVERS_LIST the ips of the pods running as jmeter remote servers
* TIMESTAMP user generated timestamp to create unique jobs

Before processing the template we get the ip of remote jmeter server runing

```oc describe pods -l app=jmeter-server-remote|grep IP|awk '{print $2'}| tr '\n' ','| sed 's/,$//'```

eg (note that the server list must be changed acconding to your environment):

```oc process -f jmeter-job-template.yaml -p TIMESTAMP=1234 -p TEST_PLAN_NAME=cool-app-jmeter.jmx -p TEST_CONFIG_MAP_NAME=jmeter-test -p REMOTE_SERVERS_LIST=172.17.0.2,172.17.0.3 -o yaml --local=true```

You get the yaml processed template.

If you want you can pipe the process command to an ```oc create -f```

eg:

```oc process -f jmeter-job-template.yam [...] |oc create -f -```

To get the status of your job you can run:

```oc get pods -l job-name=jmeter-controller-job-<TIMESTAMP>```

eg:

```oc get pods -l job-name=jmeter-controller-job-1234```

The output should be something like this:

```
NAME                               READY     STATUS      RESTARTS   AGE
jmeter-controller-job-1234-gznfw   0/1       Completed   0          1h
```

Ypu can simply grab the status with this command:

```oc get pods -l job-name=jmeter-controller-job-1234  -o=custom-columns=STATUS:.status.containerStatuses[0].state.terminated.reason --no-headers=true```

### Job from template

oc process -f bash-job-template.yaml -p JOB_SUFFIX=aNewJob -o yaml --local=true










------------------------------

--docker-image registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift


## Start server slave nodes with a test plan

unpack jmeter on your project dir

add

s2i/bin/run file

```
#!/bin/bash
echo "Before running application"
exec /usr/libexec/s2i/run
```

Create the builder

```oc new-build --binary=true --image-stream=redhat-openjdk18-openshift:1.2  --name=jmeter-remote```

build the image:

```oc start-build jmeter-remote --from-dir=./deployment --follow```

Create new app for the image built

```oc new-app jmeter-remote```


```jmeter -n -t script.jmx -R server1,server2,serverN```

You can read the official [documentation](http://jmeter.apache.org/usermanual/remote-test.html)

## Start the controller


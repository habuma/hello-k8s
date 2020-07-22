# Spring and Kubernetes - Hello World

This extremely simple example application demonstrates how to create, deploy, 
and run a Spring Boot application in a Kubernetes cluster.

## Prerequisites

You'll need...

 * A Kubernetes cluster to deploy the image and other resources to. I'll
   briefly describe how to setup a local [Kind](https://kind.sigs.k8s.io/) 
   cluster, but the application should work with any Kubernetes cluster.
 * [Docker](https://docs.docker.com/get-docker/). You'll need Docker to
   build and push the image, even if you don't use Kind.
 * A Docker repository to push the image to. Typically this means that
   you'll need an account at https://hub.docker.com. You should also be
   logged into that repository with `docker login`.
 * [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) for
   applying resources, reading logs, and general manipulation of the
   cluster.
 * [curl](https://curl.haxx.se/) or some HTTP command line tool of your 
   choice. You'll use this to make requests to the running application.

## Creating the Kind Cluster

If you choose to use a local Kind cluster, you'll need to create a new
cluster using the configuration in `src/main/k8s/kind-config.yaml`. To
do that:

```
$ kind create cluster --config src/main/k8s/kind-config.yaml
```

This configuration does two things that a simple `kind create cluster`
doesn't do:

 * Setup some extra port mappings so that once the application is running
   in the cluster, you can access it at `localhost:8000`.
 * Pin to a specific version (1.16.9) of Kubernetes. For reasons that are
   still unclear, the newest version of Kubernetes, when running in Kind,
   gives "Broken pipe" errors when attempting to read a ConfigMap or
   perform any cluster operation from the Java code. (If anyone knows how
   to make it work in newer versions of Kubernetes, I'd *LOVE* to hear about
   it.)
   
## Building and Pushing the Image

The application is a Spring Boot project. Starting with Spring Boot 2.3.0,
you can build an image from your Spring Boot project using the Maven or Gradle
Spring Boot plugin. This particular project is Maven-built, so to build an
image you would invoke Maven like this:

```
$ ./mvnw spring-boot:build-image -Dspring-boot.build-image.imageName=habuma/hello-k8s
```

Note that the `-D` parameter is optional, but if you leave it off, it will
publish the image as "library/hello-k8s". You'll probably prefer that it
publish it under your account name, so swap out "library" for your repository
username. (My DockerHub username is "habuma", so that's what I used...you'll
want to use your own username in that spot.)

If, on a different project, you are using Gradle to build a Spring Boot 2.3+
project, then you can build the image like this:

```
$ ./gradlew bootBuildImage --imageName=habuma/hello-k8s
```

As with Maven and the `-D` parameter, the `--imageName` here specifies the
image name so that it doesn't use the default.

Once the image has been built, you can run it with `docker run` if you'd like,
although you'll get an exception when it tries to read configuration from the
non-existent ConfigMap.

But the whole point of this is to run it in a Kubernetes cluster, so you'll
need to push it to DockerHub (or your repository of choice):

```
$ docker push habuma/hello-k8s
```

Again, for this step, you'll need to change "habuma" to your DockerHub username.

## Applying Prerequisite Resources to Kubernetes

Now that the image has been built and published to DockerHub, we're almost ready
to apply it in a deployment to the Kubernetes cluster. But first, a bit of setup
is required.

First, we'll need to publish a ConfigMap that contains the `greeting.message`
property that is to be injected into the `GreetingProps` bean:

```
$ kubectl apply -f src/main/k8s/configmap.yaml
```

Notice that the ConfigMap will set the `greeting.message` property to "Hello 
Kubernetes!", while the value set in `src/main/resources/application.yml` is
"Hello world!". The ConfigMap value should override the internally set value
at runtime, assuming that we have setup our cluster and ConfigMap correctly.

We'll also need to setup a Role and RoleBinding to allow the application
(by way of [Spring Cloud Kubernetes](https://spring.io/projects/spring-cloud-kubernetes) ) 
to consume the ConfigMap:

```
$ kubectl apply -f src/main/k8s/rbac.yaml
```

Failing to setup the Role and RoleBinding, will result in an exception when
the application starts that indicates that the application doesn't have
permission to consume the ConfigMap. The application will still work, but it
will fall back to the internal value of "Hello world!".

## Applying and Running the Application

Now the time has come to finally apply the deployment for our application. The
`src/main/k8s/deployment.yaml` contains the specification for the deployment,
as well as a service that our aforementioned extra port mappings can link to.
Apply it just like any other Kubernetes resource:

```
$ kubectl apply -f src/main/k8s/deployment.yaml
```

You can check in on the deployment with `kubectl get all`:

```
$ kubectl get all
NAME                                    READY   STATUS    RESTARTS   AGE
pod/hello-k8s-deploy-6f869cf477-6x9fn   1/1     Running   0          9m31s

NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)           AGE
service/hello-svc    NodePort    10.111.131.246   <none>        31234:30123/TCP   11m
service/kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP           13h

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello-k8s-deploy   2/2     2            2           11m

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/hello-k8s-deploy-6f869cf477   2         2         2       11m
``` 

Specific results will vary for you, but once you see the pod (at the top,
prefixed with `pod/hello-k8s-deploy-*`) with a status of "Running", the 
application is running. You should now be able to kick the tires on it with 
`curl`:

```
$ curl localhost:8000/hello
Hello Kubernetes!
```

If, instead of "Hello Kubernetes!", you get "Hello world!", then something is
amiss with the ConfigMap or the Role/RoleBinding. 

The application is also enabled with Spring Boot's Actuator and the `env`
endpoint is enabled. So, if you're curious to see how the ConfigMap configuration
falls in with other property sources, you can `curl` the `env` endpoint:

```
$ curl localhost:8000/actuator/env
```

This produces a *LOT* of JSON in a single line, so you might want to pipe the
results to [jq](https://stedolan.github.io/jq/) or use something other than
`curl` such as [HTTPie](https://httpie.org/) that formats the resulting JSON.

Once formatted, you should see an entry near the top that looks like this:

```
{
  "name": "bootstrapProperties-configmap.hello-k8s.default",
  "properties": {
    "greeting.message": {
      "value": "Hello Kubernetes!"
    }
  }
},
```

This is the property source provided by Spring Cloud Kubernetes that consumes
the ConfigMap.

A little lower, you'll see one that looks like this:

```
{
  "name": "applicationConfig: [classpath:/application.yml]",
  "properties": {
    "spring.application.name": {
      "value": "hello-k8s",
      "origin": "class path resource [application.yml]:3:11"
    },
    "greeting.message": {
      "value": "Hello world!",
      "origin": "class path resource [application.yml]:6:12"
    },
    "management.endpoints.web.exposure.include[0]": {
      "value": "info",
      "origin": "class path resource [application.yml]:13:11"
    },
    "management.endpoints.web.exposure.include[1]": {
      "value": "health",
      "origin": "class path resource [application.yml]:14:11"
    },
    "management.endpoints.web.exposure.include[2]": {
      "value": "env",
      "origin": "class path resource [application.yml]:15:11"
    }
  }
},
```

Notice that the `spring.application.name` is "hello-k8s". This matches
the `metadata.name` property set in `src/main/k8s/configmap.yaml`, which
is what links this application to that particular ConfigMap.

This is the property source that comes from the internal configuration 
in `src/main/resources/application.yml`. Because the ConfigMap property
source is higher, it takes higher precedence. Had the ConfigMap not
been created, had the Role/RoleBinding not been created, or had the
ConfigMap not been readable for any other reason, the internal configuration
would have been used instead.


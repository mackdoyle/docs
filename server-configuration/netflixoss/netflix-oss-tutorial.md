
[Source](http://callistaenterprise.se/blogg/teknik/2015/04/10/building-microservices-with-spring-cloud-and-netflix-oss-part-1/ "Permalink to Building microservices with Spring Cloud and Netflix OSS, part 1")

# Building microservices with Spring Cloud and Netflix OSS, part 1

![Foto av Magnus Larsson][1]

###  10 April 2015  // [Magnus Larsson][2]

In the [previous blog post][3] we defined an operations model for usage of microservices. In this blog post we will start to look at how we can implement the model using [Spring Cloud][4] and [Netflix OSS][5]. From the operations model we will cover the core parts: _service discovery_, _dynamic routing_, _load balancing_ and to some extend an _edge server_, leaving the other parts to upcoming blog posts.

We will use some core components from Spring Cloud and Netflix OSS to allow separately deployed microservices to communicate with each other with no manual administration required, e.g. keeping track of what ports each microservice use or manual configuration of routing rules. To avoid problems with port conflicts, our microservices will dynamically allocate free ports from a port range at startup. To allow simple access to the microservices we will use an edge server that provides a well known entry point to the microservice landscape.

After a quick introduction of the Spring Cloud and Netflix OSS components we will present a system landscape that we will use throughout the blog series. We will go through how to access the source code and build it. We will also make a brief walkthrough of the source code pointing out the most important parts. Finally we wrap up by running through some test cases on how to access the services and also demonstrate how simple it is to bring up a new instance of a microservice and get the load balancer to start to use it, again without any manual configuration required.

## 1\. Spring Cloud and Netflix OSS

Spring Cloud is a [new project][6] in the [spring.io family][7] with a set of components that can be used to implement our operations model. To a large extent Spring Cloud 1.0 is based on components from [Netflix OSS][5]. Spring Cloud integrates the Netflix components in the Spring environment in a very nice way using auto configuration and convention over configuration similar to how [Spring Boot][8] works.

The table below maps the generic components in the [operations model][9] to the actual components that we will use throughout this blog series:

![][10]

In this blog post we will cover _Eureka_, _Ribbon_ and to some extent _Zuul_:

* **Netflix Eureka** \- Service Discovery Server  
Netflix Eureka allows microservices to register themselves at runtime as they appear in the system landscape.

* **Netflix Ribbon** \- Dynamic Routing and Load Balancer  
Netflix Ribbon can be used by service consumers to lookup services at runtime. Ribbon uses the information available in Eureka to locate appropriate service instances. If more than one instance is found, Ribbon will apply load balancing to spread the requests over the available instances. Ribbon does not run as a separate service but instead as an embedded component in each service consumer.

* **Netflix Zuul** \- Edge Server  
Zuul is (of course) our [gatekeeper][11] to the outside world, not allowing any unauthorized external requests pass through. Zulu also provides a well known entry point to the microservices in the system landscape. Using dynamically allocated ports is convenient to avoid port conflicts and to minimize administration but it makes it of course harder for any given service consumer. Zuul uses Ribbon to lookup available services and routes the external request to an appropriate service instance. In this blog post we will only use Zuul to provide a well known entry point, leaving the security aspects for coming blog posts.

**Note:** The microservices that are allowed to be accessed externally through the edge server can be seen as an [API][12] to the system landscape.

## 2\. A system landscape

To be able to test the components we need a system landscape to play with. For the scope of this blog post we have developed a landscape that looks like:

![system-landscape][13]

It contains four business services (_green boxes_):

* Three core services responsible for handling information regarding **products**, **recommendations** and **reviews**.
* One composite service, **product-composite**, that can aggregate information from the three core services and compose a view of product information together with reviews and recommendations of a product.

To support the business services we use the following infrastructure services and components (_blue boxes_):

* **Service Discovery Server** (Netflix Eureka)
* **Dynamic Routing and Load Balancer** (Netflix Ribbon)
* **Edge Server** (Netflix Zuul)

&gt; To emphasize the differences between microservices and monolithic applications we will run each service in a separate microservice, i.e. in separate processes. In a large scale system landscape it will most probably be inconvenient with this type of fine grained microservices. Instead, a number of related services will probably be grouped in one and the same microservice to keep the number of microservices at a manageable level. But still without falling back into huge monolithic applications…

## 3\. Build from source

If you want to check out the source code and test it on your own you need to have Java SE 8 and Git installed. Then perform:

    $ git clone https://github.com/callistaenterprise/blog-microservices.git
    $ cd blog-microservices
    $ git checkout -b B1 M1.1

This will result in the following structure of components:

![source-code][14]

Each component is built separately (remember that we no longer are building monolithic applications :-) so each component have their own build file. We use [Gradle][15] as the build system, if you don't have Gradle installed the build file will download it for you. To simplify the process we have a small shell script that you can use to build the components:

    $ ./build-all.sh

&gt; If you are on **Windows** you can execute the corresponding bat-file `build-all.bat`!

It should result in six log messages that all says:

    BUILD SUCCESSFUL

## 4\. Source code walkthrough

Let's take a quick look at some key source code construct. Each microservice is developed as standalone [Spring Boot][8] application and uses [Undertow][16], a lightweight Servlet 3.1 container, as its web server. [Spring MVC][17] is used to implement the REST based services and [Spring RestTemplate][18] is used to perform outgoing calls. If you want to know more about these core technologies you can for example take a look at the following [blog post][19].

Instead let us focus on how to use the functionality in Spring Cloud and Netflix OSS!

**Note:** We have intentionally made the implementation as simple as possible to make the source code easy to grasp and understand.

### 4.1 Gradle dependencies

In the spirit of Spring Boot, Spring Cloud has defined a set of starter dependencies making it very easy to bring in the dependencies you need for a specific feature. To use Eureka and Ribbon in a microservice to register and/or call other services simply add the following to the build file:

        compile("org.springframework.cloud:spring-cloud-starter-eureka:1.0.0.RELEASE")

For a complete example see [product-service/build.gradle][20].

To be able to setup an Eureka server add the following dependency:

        compile('org.springframework.cloud:spring-cloud-starter-eureka-server:1.0.0.RELEASE')

For a complete example see [discovery-server/build.gradle][21].

### 4.2. Infrastructure servers

Setting up an infrastructure server based on Spring Cloud and Netflix OSS is really easy. E.g. for a Eureka server add the annotation `@EnableEurekaServer` to a standard Spring Boot application:

    @SpringBootApplication
    @EnableEurekaServer
    public class EurekaApplication {

        public static void main(String[] args) {
            SpringApplication.run(EurekaApplication.class, args);
        }
    }

For a complete example see [EurekaApplication.java][22].

To bring up a Zuul server you add a `@EnableZuulProxy` \- annotation instead. For a complete example see [ZuulApplication.java][23].

With these simple annotations you get a default server configurations that gets yo started. When needed, the default configurations can be overridden with specific settings. One example of overriding the default configuration is where we have limited what services that the edge server is allowed to route calls to. By default Zuul set up a route to every service it can find in Eureka. With the following configuration in the `application.yml` \- file we have limited the routes to only allow calls to the composite product service:

    zuul:
      ignoredServices: "*"
      routes:
        productcomposite:
          path: /productcomposite/**

For a complete example see [edge-server/application.yml][24].

### 4.3 Business services

To auto register microservices with Eureka, add a `@EnableDiscoveryClient` \- annotation to the Spring Boot application.

    @SpringBootApplication
    @EnableDiscoveryClient
    public class ProductServiceApplication {

        public static void main(String[] args) {
            SpringApplication.run(ProductServiceApplication.class, args);
        }
    }

For a complete example see [ProductServiceApplication.java][25].

To lookup and call an instance of a microservice, use Ribbon and for example a Spring RestTemplate like:

        @Autowired
        private LoadBalancerClient loadBalancer;
        ...
        public ResponseEntity<list<recommendation>&gt; getReviews(int productId) {

                ServiceInstance instance = loadBalancer.choose("review");
                URI uri = instance.getUri();
    		...
                response = restTemplate.getForEntity(url, String.class);

The service consumer only need to know about the name of the service (`review` in the example above), Ribbon (i.e. the `LoadBalancerClient` class) will find a service instance and return its URI to the service consumer.

For a complete example see [Util.java][26] and [ProductCompositeIntegration.java][27].

## 5\. Start up the system landscape

&gt; In this blog post we will start the microservices as java processes in our local development environment. In followup blog posts we will describe how to deploy microservices to both cloud infrastructures and Docker containers!

To be able to run some of the commands used below you need to have the tools [cURL][28] and [jq][29] installed.

Each microservice is started using the command `./gradlew bootRun`.

First start the infrastructure microservices, e.g.:

    $ cd .../blog-microservices/microservices

    $ cd support/discovery-server;  ./gradlew bootRun
    $ cd support/edge-server;       ./gradlew bootRun

Once they are started up, launch the business microservices:

    $ cd core/product-service;                ./gradlew bootRun
    $ cd core/recommendation-service;         ./gradlew bootRun
    $ cd core/review-service;                 ./gradlew bootRun
    $ cd composite/product-composite-service; ./gradlew bootRun

&gt; If you are on **Windows** you can execute the corresponding bat-file `start-all.bat`!

Once the microservices are started up and registered with the service discovery server they should write the following in the log:

    DiscoveryClient ... - registration status: 204

In the service discovery web app we should now be able to see our four business services and the edge server (<http: localhost:8761="">):

![Eureka][30]

To find out more details about our services, e.g. what ip addresses and ports they use, we can use the Eureka REST API, e.g.:

    $ curl -s -H "Accept: application/json" http://localhost:8761/eureka/apps | jq '.applications.application[] | {service: .name, ip: .instance.ipAddr, port: .instance.port."$"}'
    {
      "service": "PRODUCT",
      "ip": "192.168.0.116",
      "port": "59745"
    }
    {
      "service": "REVIEW",
      "ip": "192.168.0.116",
      "port": "59178"
    }
    {
      "service": "RECOMMENDATION",
      "ip": "192.168.0.116",
      "port": "48014"
    }
    {
      "service": "PRODUCTCOMPOSITE",
      "ip": "192.168.0.116",
      "port": "51658"
    }
    {
      "service": "EDGESERVER",
      "ip": "192.168.0.116",
      "port": "8765"
    }

We are now ready to try some test cases, first some happy days tests to verify that we can reach our services and then we will se how we can bring up a new instance of a microservice and get Ribbon to load balance requests over multiple instances of that service.

**Note:** In coming blog posts we will also try failure scenarios to demonstrate how a circuit breaker works.

### 5.1 Happy days

Start to call the composite service through the edge server, The edge server is found on the port 8765 (see its application.yml file) and as we have seen above we can use the path `/productcomposite/**` to reach the product-composite service through our edge server. The should return a composite response (shortened for brevity) like:

    $ curl -s localhost:8765/productcomposite/product/1 | jq .
    {
        "name": "name",
        "productId": 1,
        "recommendations": [
            {
                "author": "Author 1",
                "rate": 1,
                "recommendationId": 1
            },
            ...
        ],
        "reviews": [
            {
                "author": "Author 1",
                "reviewId": 1,
                "subject": "Subject 1"
            },
            ...
        ],
        "weight": 123
    }

…if we are on the inside of the microservice landscape we can actually call the services directly without going through the edge server. The problem is of course that we don't know on what ports the services runs on since they are allocated dynamically. But if we look at the output from the call to the eureka rest api we can find out what ports they listen to. To call the composite service and then the three core services we should be able to use the following commands (given the port information in the response from the call to the eureka rest api above)

    $ curl -s localhost:51658/product/1 | jq .
    $ curl -s localhost:59745/product/1 | jq .
    $ curl -s localhost:59178/review?productId=1 | jq .
    $ curl -s localhost:48014/recommendation?productId=1 | jq .

Try it out in your own environment!

### 5.2 Dynamic load balancing

To avoid service outage due to a failing service or temporary network problems it is very common to have more than one service instance of the same type running and using a load balancer to spread the incoming calls over the instances. Since we are using dynamic allocated ports and a service discovery server it is very easy to add a new instance. For example simply start a new review service and it will allocate a new port dynamically and register itself to the service discovery server.

    $ cd .../blog-microservices/microservices/core/review-service
    $ ./gradlew bootRun

After a short while you can see the second instance in the service discovery web app (<http: localhost:8761="">):

![eureka][31]

If you run the previous curl command (`curl -s localhost:8765/productcomposite/product/1 | jq .`) a couple of times and look into the logs of the two review instances you will see how the load balancer automatically spread in calls over the two instances without any manual configuration required:

![Hystrix][32]

## 6\. Summary

We have seen how Spring Cloud and Netflix OSS components can be used to simplify making separately deployed microservices to work together with no manual administration in terms of keeping track of what ports each microservice use or manual configuration of routing rules. When new instances are started they are automatically detected by the load balancer through the service discovery server and can start to receive requests. Using the edge server we can control what microservices that are exposed externally, establishing av API for the system landscape.


[Source](http://callistaenterprise.se/blogg/teknik/2015/04/15/building-microservices-with-spring-cloud-and-netflix-oss-part-2/ "Permalink to Building microservices with Spring Cloud and Netflix OSS, part 2")

# Building microservices with Spring Cloud and Netflix OSS, part 2

###  15 April 2015  // [Magnus Larsson][2]

In [Part 1][3] we used core components in [Spring Cloud][4] and [Netflix OSS][5], i.e. _Eureka_, _Ribbon_ and _Zuul_, to partially implement our [operations model][6], enabling separately deployed microservices to communicate with each other. In this blog post we will focus on fault handling in a microservice landscape, improving resilience using _Hystrix_, Netflix circuit breaker.

Now bad things will start to happen in the system landscape that we established in [Part 1][3]. Some of the core services that the composite service depends on will suddenly not respond, jeopardizing the composite service if faults are not handled correctly.

In general we call this type of problem a _chain of failures_, where an error in one component can cause errors to occur in other components that depend on the failing component. This needs special attention in a microservice based system landscape where, potentially a large number of, separately deployed microservices communicate with each other. One common solution to this problem is to apply a _circuit breaker_ pattern, for details see the book [Release It!][7] or read the blog post [Fowler - Circuit Breaker][8]. A _circuit breaker_ typically applies state transitions like:

![][9]  
(**Source:** [Release It!][7])

Enter _Netflix Hystrix_ and some powerful annotations provided by _Spring Cloud_!

## 1\. Spring Cloud and Netflix OSS

From the table presented in [Part 1][3] we will cover: _Hystrix_, _Hystrix dashboard_ and _Turbine_.

![][10]

* **Netflix Hystrix** \- Circuit breaker  
Netflix Hystrix provides circuit breaker capabilities to a service consumer. If a service doesn't respond (e.g. due to a timeout or a communication error), Hystrix can redirect the call to an internal fallback method in the service consumer. If a service repeatedly fails to respond, Hystrix will open the circuit and fast fail (i.e. call the internal fallback method without trying to call the service) on every subsequent call until the service is available again. To determine wether the service is available again Hystrix allow some requests to try out the service even if the circuit is open. Hystrix executes embedded within its service consumer.

* **Netflix Hystrix dashboard and Netflix Turbine** \- Monitor Dashboard  
Hystrix dashboard can be used to provide a graphical overview of circuit breakers and Turbine can, based on information in Eureka, provide the dashboard with information from all circuit breakers in a system landscape. A sample screenshot from Hystrix dashboard and Turbine in action:

![Hystrix][1100]

## 2\. The system landscape

The system landscape from [Part 1][3] is complemented with supporting infrastructure servers for _Hystrix dashboard_ and _Turbine_. The service _product-composite_ is also enhanced with a _Hystrix_ based circuit breaker. The two new components are marked with a red line in the updated picture below:

![system-landscape][12]

&gt; As in [Part 1][3], we emphasize the differences between microservices and monolithic applications by running each service in a separate microservice, i.e. in separate processes.

## 3\. Build from source

As in [Part 1][3] we use Java SE 8, Git and Gradle. So, to access the source code and build it perform:

    $ git clone https://github.com/callistaenterprise/blog-microservices.git
    $ cd blog-microservices
    $ git checkout -b B2 M2.1
    $ ./build-all.sh

&gt; If you are on **Windows** you can execute the corresponding bat-file `build-all.bat`!

Two new source code components have been added since [Part 1][3]: _monitor-dashboard_ and _turbine_:

![source-code][1300]

The build should result in eight log messages that all says:

    BUILD SUCCESSFUL

## 4\. Source code walkthrough

New from [Part 1][3] is the use of Hystrix as a circuit breaker in the microservice product-composite, so we will focus on the additional source code required to put the circuit breaker in place.

### 4.1 Gradle dependencies

We now have a couple of Hystrix-based starter dependencies to drag into our build files. Since Hysteric use [RabbitMQ][14] to communicate between circuit breakers and dashboards we also need to setup dependencies for that as well.

For a service consumer, that want to use Hystrix as a circuit breaker, we need to add:

        compile("org.springframework.cloud:spring-cloud-starter-hystrix:1.0.0.RELEASE")
        compile("org.springframework.cloud:spring-cloud-starter-bus-amqp:1.0.0.RELEASE")
        compile("org.springframework.cloud:spring-cloud-netflix-hystrix-amqp:1.0.0.RELEASE")

For a complete example see [product-composite-service/build.gradle][15].

To be able to setup an Turbine server add the following dependency:

        compile('org.springframework.cloud:spring-cloud-starter-turbine-amqp:1.0.0.RELEASE')

For a complete example see [turbine/build.gradle][16].

### 4.2. Infrastructure servers

Set up a Turbine server by adding the annotation `@EnableTurbineAmqp` to a standard Spring Boot application:

    @SpringBootApplication
    @EnableTurbineAmqp
    @EnableDiscoveryClient
    public class TurbineApplication {

        public static void main(String[] args) {
            SpringApplication.run(TurbineApplication.class, args);
        }

    }

For a complete example see [TurbineApplication.java][17].

To setup a Hystrix Dashboard add the annotation `@EnableHystrixDashboard` instead. For a complete example see [HystrixDashboardApplication.java][18].

With these simple annotation you get a default server configuration that makes you get started. When needed, the default configurations can be overridden with specific settings.

### 4.3 Business services

To enable Hystrix, add a `@EnableCircuitBreaker`-annotation to your Spring Boot application. To actually put Hystrix in action, annotate the method that Hystrix shall monitor with `@HystrixCommand` where we also can specify a fallback-method, e.g.:

        @HystrixCommand(fallbackMethod = "defaultReviews")
        public ResponseEntity<list<review>&gt; getReviews(int productId) {
            ...
        }

        public ResponseEntity<list<review>&gt; defaultReviews(int productId) {
            ...
        }

The fallback method is used by Hystrix in case of an error (call to the service fails or a timeout occurs) or to fast fail if the circuit is open. For a complete example see [ProductCompositeIntegration.java][19].

## 5\. Start up the system landscape

&gt; As in [Part 1][3], we will start the microservices as java processes in our local development environment and you need to have the [cURL][20] and [jq][21] tools installed to be able to run some of the commands below.

As already mentioned, Hystrix use RabbitMQ for internal communication so we need to have it installed and running before we can start up our system landscape. Follow the instructions at [Downloading and Installing][22]. Then start RabbitMQ use the `rabbitmq-server` program in the `sbin`-folder of your installation.

    $ ~/Applications/rabbitmq_server-3.4.3/sbin/rabbitmq-server

                  RabbitMQ 3.4.3. Copyright (C) 2007-2014 GoPivotal, Inc.
      ##  ##      Licensed under the MPL.  See http://www.rabbitmq.com/
      ##  ##
      ##########  Logs: /Users/magnus/Applications/rabbitmq_server-3.4.3/sbin/../var/log/rabbitmq/rabbit@Magnus-MacBook-Pro.log
      ######  ##        /Users/magnus/Applications/rabbitmq_server-3.4.3/sbin/../var/log/rabbitmq/rabbit@Magnus-MacBook-Pro-sasl.log
      ##########
                  Starting broker... completed with 6 plugins.

&gt; If you are on **Windows** use Windows Services to ensure that the RabbitMQ service is started!

We are now ready to start up the system landscape. Each microservice is started using the command `./gradlew bootRun`.

First start the infrastructure microservices, e.g.:

    $ cd support/discovery-server;  ./gradlew bootRun
    $ cd support/edge-server;       ./gradlew bootRun
    $ cd support/monitor-dashboard; ./gradlew bootRun
    $ cd support/turbine;           ./gradlew bootRun

Once they are started up, launch the business microservices:

    $ cd core/product-service;                ./gradlew bootRun
    $ cd core/recommendation-service;         ./gradlew bootRun
    $ cd core/review-service;                 ./gradlew bootRun
    $ cd composite/product-composite-service; ./gradlew bootRun

&gt; If you are on **Windows** you can execute the corresponding bat-file `start-all.bat`!

Once the microservices are started up and registered with the service discovery server they should write the following in the log:

    DiscoveryClient ... - registration status: 204

As in [Part 1][3], we should be able to see our four business services and the edge-server in the service discovery web app (<http: localhost:8761="">):

![Eureka][23]

Finally ensure that the circuit breakers are operational, e.g. _closed_. Try a call to the composite service through the edge-server as in [Part 1][3] (shortened for brevity:

    $ curl -s localhost:8765/productcomposite/product/1 | jq .
    {
        "name": "name",
        "productId": 1,
        "recommendations": [
            {
                "author": "Author 1",
                "rate": 1,
                "recommendationId": 0
            },
            ...
        ],
        "reviews": [
            {
                "author": "Author 1",
                "reviewId": 1,
                "subject": "Subject 1"
            },
            ...
        ],
        "weight": 123
    }

Go to the url <http: localhost:7979=""> in a web browser, enter the url <http: localhost:8989="" turbine.stream=""> and click on the "_Monitor Stream_" – button):

![Hystrix][24]

We can see that the composite service have three circuit breakers operational, one for each core service it depends on. They are all fine, i.e. _closed_. We are now ready to try out a negative test to see the circuit breaker in action!

## 6\. Something goes wrong

Stop the `review` microservice and retry the command above:

    $ curl -s localhost:8765/productcomposite/product/1 | jq .
    {
        "name": "name",
        "productId": 1,
        "recommendations": [
            {
                "author": "Author 1",
                "rate": 1,
                "recommendationId": 0
            },
            ...
        ],
        "reviews": null,
        "weight": 123
    }

The **review** \- part of the response is now empty, but the rest of the reply remains intact! Look at the log of the `product-composite` services and you will find warnings:

    2015-04-02 15:13:36.344  INFO 29901 --- [eIntegration-10] s.c.m.c.p.s.ProductCompositeIntegration  : GetRecommendations...
    2015-04-02 15:13:36.497  INFO 29901 --- [teIntegration-2] s.c.m.c.p.s.ProductCompositeIntegration  : GetReviews...
    2015-04-02 15:13:36.498  WARN 29901 --- [teIntegration-2] s.c.m.composite.product.service.Util     : Failed to resolve serviceId 'review'. Fallback to URL 'http://localhost:8081/review'.
    2015-04-02 15:13:36.500  WARN 29901 --- [teIntegration-2] s.c.m.c.p.s.ProductCompositeIntegration  : Using fallback method for review-service

I.e. the circuit breaker has detected a problem with the `review` service and routed the caller to the fallback method in the service consumer. In this case we simply return null but we could, for example, return data from a local cache to provide a best effort result when the `review` service is unavailable.

The circuit is still closed since the error is not that frequent:

![Hystrix][25]

Let's increase the error frequency over the limits where Hystrix will open the circuit and start to fast fail (we use Hystrix default values to keep the blog post short…). We use the [Apache HTTP server benchmarking tool][26] for this:

    ab -n 30 -c 5 localhost:8765/productcomposite/product/1

Now the circuit will be opened:

![Hystrix][27]

…and subsequent calls will fast fail, i.e. the circuit breaker will redirect the caller directly to its fallback method without trying to get the reviews from the `review` service. The log will no longer contain a message that says `GetReviews...`:

    2015-04-02 15:14:03.930  INFO 29901 --- [teIntegration-5] s.c.m.c.p.s.ProductCompositeIntegration  : GetRecommendations...
    2015-04-02 15:14:03.984  WARN 29901 --- [ XNIO-2 task-62] s.c.m.c.p.s.ProductCompositeIntegration  : Using fallback method for review-service

However, from time to time it will let some calls pass through to see if they succeeds, i.e. to see if the `review` service is available again. We can see that by repeating the curl call a number of times and look in the log of the `product-composite` service:

    2015-04-02 15:17:33.587  INFO 29901 --- [eIntegration-10] s.c.m.c.p.s.ProductCompositeIntegration  : GetRecommendations...
    2015-04-02 15:17:33.769  INFO 29901 --- [eIntegration-10] s.c.m.c.p.s.ProductCompositeIntegration  : GetReviews...
    2015-04-02 15:17:33.769  WARN 29901 --- [eIntegration-10] s.c.m.composite.product.service.Util     : Failed to resolve serviceId 'review'. Fallback to URL 'http://localhost:8081/review'.
    2015-04-02 15:17:33.770  WARN 29901 --- [eIntegration-10] s.c.m.c.p.s.ProductCompositeIntegration  : Using fallback method for review-service
    2015-04-02 15:17:34.431  INFO 29901 --- [eIntegration-10] s.c.m.c.p.s.ProductCompositeIntegration  : GetRecommendations...
    2015-04-02 15:17:34.569  WARN 29901 --- [ XNIO-2 task-18] s.c.m.c.p.s.ProductCompositeIntegration  : Using fallback method for review-service
    2015-04-02 15:17:35.209  INFO 29901 --- [eIntegration-10] s.c.m.c.p.s.ProductCompositeIntegration  : GetRecommendations...
    2015-04-02 15:17:35.402  WARN 29901 --- [ XNIO-2 task-20] s.c.m.c.p.s.ProductCompositeIntegration  : Using fallback method for review-service
    2015-04-02 15:17:36.043  INFO 29901 --- [eIntegration-10] s.c.m.c.p.s.ProductCompositeIntegration  : GetRecommendations...
    2015-04-02 15:17:36.192  WARN 29901 --- [ XNIO-2 task-21] s.c.m.c.p.s.ProductCompositeIntegration  : Using fallback method for review-service
    2015-04-02 15:17:36.874  INFO 29901 --- [eIntegration-10] s.c.m.c.p.s.ProductCompositeIntegration  : GetRecommendations...
    2015-04-02 15:17:37.031  WARN 29901 --- [ XNIO-2 task-22] s.c.m.c.p.s.ProductCompositeIntegration  : Using fallback method for review-service
    2015-04-02 15:17:41.148  INFO 29901 --- [eIntegration-10] s.c.m.c.p.s.ProductCompositeIntegration  : GetRecommendations...
    2015-04-02 15:17:41.340  INFO 29901 --- [eIntegration-10] s.c.m.c.p.s.ProductCompositeIntegration  : GetReviews...
    2015-04-02 15:17:41.340  WARN 29901 --- [eIntegration-10] s.c.m.composite.product.service.Util     : Failed to resolve serviceId 'review'. Fallback to URL 'http://localhost:8081/review'.
    2015-04-02 15:17:41.341  WARN 29901 --- [eIntegration-10] s.c.m.c.p.s.ProductCompositeIntegration  : Using fallback method for review-service

As we can see from the log output, every fifth call is allowed to try to call the `review` service (still without success…).

Let's start the `review` service again and try calling the composite service!

**Note:** You might need to be a bit patient here (max 1 min), both the service discovery server (Eureka) and the dynamic router (Ribbon) must be made aware of that a `review` service instance is available again before the call succeeds.

Now we can see that the response is ok, i.e. the review part is back in the response, and the circuit is closed again:

![Hystrix][280]

## 7\. Summary

We have seen how _Netflix Hystrix_ can be used as a _circuit breaker_ to efficiently handle the problem with _chain of failures_, i.e. where a failing microservice potentially can cause a system outage of a large part of a microservice landscape due to propagating errors. Thanks to the annotations and starter dependencies available in Spring Cloud it is very easy to get started with Hystrix in a Spring environment. Finally the dashboard capabilities provided by Hystrix dashboard and Turbine makes it possible to monitor a large number of circuit breakers in a system landscape.

----

# Building microservices with Spring Cloud and Netflix OSS, part 2

###  15 April 2015  // [Magnus Larsson][200]

In [Part 1][300] we used core components in [Spring Cloud][400] and [Netflix OSS][500], i.e. _Eureka_, _Ribbon_ and _Zuul_, to partially implement our [operations model][600], enabling separately deployed microservices to communicate with each other. In this blog post we will focus on fault handling in a microservice landscape, improving resilience using _Hystrix_, Netflix circuit breaker.

Now bad things will start to happen in the system landscape that we established in [Part 1][300]. Some of the core services that the composite service depends on will suddenly not respond, jeopardizing the composite service if faults are not handled correctly.

In general we call this type of problem a _chain of failures_, where an error in one component can cause errors to occur in other components that depend on the failing component. This needs special attention in a microservice based system landscape where, potentially a large number of, separately deployed microservices communicate with each other. One common solution to this problem is to apply a _circuit breaker_ pattern, for details see the book [Release It!][7] or read the blog post [Fowler - Circuit Breaker][8]. A _circuit breaker_ typically applies state transitions like:

![][9]  
(**Source:** [Release It!][700])

Enter _Netflix Hystrix_ and some powerful annotations provided by _Spring Cloud_!

## 1\. Spring Cloud and Netflix OSS

From the table presented in [Part 1][3] we will cover: _Hystrix_, _Hystrix dashboard_ and _Turbine_.

![][1000]

* **Netflix Hystrix** \- Circuit breaker  
Netflix Hystrix provides circuit breaker capabilities to a service consumer. If a service doesn't respond (e.g. due to a timeout or a communication error), Hystrix can redirect the call to an internal fallback method in the service consumer. If a service repeatedly fails to respond, Hystrix will open the circuit and fast fail (i.e. call the internal fallback method without trying to call the service) on every subsequent call until the service is available again. To determine wether the service is available again Hystrix allow some requests to try out the service even if the circuit is open. Hystrix executes embedded within its service consumer.

* **Netflix Hystrix dashboard and Netflix Turbine** \- Monitor Dashboard  
Hystrix dashboard can be used to provide a graphical overview of circuit breakers and Turbine can, based on information in Eureka, provide the dashboard with information from all circuit breakers in a system landscape. A sample screenshot from Hystrix dashboard and Turbine in action:

![Hystrix][1100]

## 2\. The system landscape

The system landscape from [Part 1][300] is complemented with supporting infrastructure servers for _Hystrix dashboard_ and _Turbine_. The service _product-composite_ is also enhanced with a _Hystrix_ based circuit breaker. The two new components are marked with a red line in the updated picture below:

![system-landscape][1200]

&gt; As in [Part 1][300], we emphasize the differences between microservices and monolithic applications by running each service in a separate microservice, i.e. in separate processes.

## 3\. Build from source

As in [Part 1][3] we use Java SE 8, Git and Gradle. So, to access the source code and build it perform:

    $ git clone https://github.com/callistaenterprise/blog-microservices.git
    $ cd blog-microservices
    $ git checkout -b B2 M2.1
    $ ./build-all.sh

&gt; If you are on **Windows** you can execute the corresponding bat-file `build-all.bat`!

Two new source code components have been added since [Part 1][3]: _monitor-dashboard_ and _turbine_:

![source-code][1300]

The build should result in eight log messages that all says:

    BUILD SUCCESSFUL

## 4\. Source code walkthrough

New from [Part 1][3] is the use of Hystrix as a circuit breaker in the microservice product-composite, so we will focus on the additional source code required to put the circuit breaker in place.

### 4.1 Gradle dependencies

We now have a couple of Hystrix-based starter dependencies to drag into our build files. Since Hysteric use [RabbitMQ][1400] to communicate between circuit breakers and dashboards we also need to setup dependencies for that as well.

For a service consumer, that want to use Hystrix as a circuit breaker, we need to add:

        compile("org.springframework.cloud:spring-cloud-starter-hystrix:1.0.0.RELEASE")
        compile("org.springframework.cloud:spring-cloud-starter-bus-amqp:1.0.0.RELEASE")
        compile("org.springframework.cloud:spring-cloud-netflix-hystrix-amqp:1.0.0.RELEASE")

For a complete example see [product-composite-service/build.gradle][1500].

To be able to setup an Turbine server add the following dependency:

```java
        compile('org.springframework.cloud:spring-cloud-starter-turbine-amqp:1.0.0.RELEASE')
```

For a complete example see [turbine/build.gradle][1600].

### 4.2. Infrastructure servers

Set up a Turbine server by adding the annotation `@EnableTurbineAmqp` to a standard Spring Boot application:

    @SpringBootApplication
    @EnableTurbineAmqp
    @EnableDiscoveryClient
    public class TurbineApplication {

        public static void main(String[] args) {
            SpringApplication.run(TurbineApplication.class, args);
        }

    }

For a complete example see [TurbineApplication.java][1700].

To setup a Hystrix Dashboard add the annotation `@EnableHystrixDashboard` instead. For a complete example see [HystrixDashboardApplication.java][1800].

With these simple annotation you get a default server configuration that makes you get started. When needed, the default configurations can be overridden with specific settings.

### 4.3 Business services

To enable Hystrix, add a `@EnableCircuitBreaker`-annotation to your Spring Boot application. To actually put Hystrix in action, annotate the method that Hystrix shall monitor with `@HystrixCommand` where we also can specify a fallback-method, e.g.:

        @HystrixCommand(fallbackMethod = "defaultReviews")
        public ResponseEntity<list<review>&gt; getReviews(int productId) {
            ...
        }

        public ResponseEntity<list<review>&gt; defaultReviews(int productId) {
            ...
        }

The fallback method is used by Hystrix in case of an error (call to the service fails or a timeout occurs) or to fast fail if the circuit is open. For a complete example see [ProductCompositeIntegration.java][1900].

## 5\. Start up the system landscape

&gt; As in [Part 1][3], we will start the microservices as java processes in our local development environment and you need to have the [cURL][2000] and [jq][21000] tools installed to be able to run some of the commands below.

As already mentioned, Hystrix use RabbitMQ for internal communication so we need to have it installed and running before we can start up our system landscape. Follow the instructions at [Downloading and Installing][220]. Then start RabbitMQ use the `rabbitmq-server` program in the `sbin`-folder of your installation.

    $ ~/Applications/rabbitmq_server-3.4.3/sbin/rabbitmq-server

                  RabbitMQ 3.4.3. Copyright (C) 2007-2014 GoPivotal, Inc.
      ##  ##      Licensed under the MPL.  See http://www.rabbitmq.com/
      ##  ##
      ##########  Logs: /Users/magnus/Applications/rabbitmq_server-3.4.3/sbin/../var/log/rabbitmq/rabbit@Magnus-MacBook-Pro.log
      ######  ##        /Users/magnus/Applications/rabbitmq_server-3.4.3/sbin/../var/log/rabbitmq/rabbit@Magnus-MacBook-Pro-sasl.log
      ##########
                  Starting broker... completed with 6 plugins.

&gt; If you are on **Windows** use Windows Services to ensure that the RabbitMQ service is started!

We are now ready to start up the system landscape. Each microservice is started using the command `./gradlew bootRun`.

First start the infrastructure microservices, e.g.:

    $ cd support/discovery-server;  ./gradlew bootRun
    $ cd support/edge-server;       ./gradlew bootRun
    $ cd support/monitor-dashboard; ./gradlew bootRun
    $ cd support/turbine;           ./gradlew bootRun

Once they are started up, launch the business microservices:

    $ cd core/product-service;                ./gradlew bootRun
    $ cd core/recommendation-service;         ./gradlew bootRun
    $ cd core/review-service;                 ./gradlew bootRun
    $ cd composite/product-composite-service; ./gradlew bootRun

&gt; If you are on **Windows** you can execute the corresponding bat-file `start-all.bat`!

Once the microservices are started up and registered with the service discovery server they should write the following in the log:

    DiscoveryClient ... - registration status: 204

As in [Part 1][3], we should be able to see our four business services and the edge-server in the service discovery web app (<http: localhost:8761="">):

![Eureka][2300]

Finally ensure that the circuit breakers are operational, e.g. _closed_. Try a call to the composite service through the edge-server as in [Part 1][3] (shortened for brevity:

    $ curl -s localhost:8765/productcomposite/product/1 | jq .
    {
        "name": "name",
        "productId": 1,
        "recommendations": [
            {
                "author": "Author 1",
                "rate": 1,
                "recommendationId": 0
            },
            ...
        ],
        "reviews": [
            {
                "author": "Author 1",
                "reviewId": 1,
                "subject": "Subject 1"
            },
            ...
        ],
        "weight": 123
    }

Go to the url <http: localhost:7979=""> in a web browser, enter the url <http: localhost:8989="" turbine.stream=""> and click on the "_Monitor Stream_" – button):

![Hystrix][2400]

We can see that the composite service have three circuit breakers operational, one for each core service it depends on. They are all fine, i.e. _closed_. We are now ready to try out a negative test to see the circuit breaker in action!

## 6\. Something goes wrong

Stop the `review` microservice and retry the command above:

    $ curl -s localhost:8765/productcomposite/product/1 | jq .
    {
        "name": "name",
        "productId": 1,
        "recommendations": [
            {
                "author": "Author 1",
                "rate": 1,
                "recommendationId": 0
            },
            ...
        ],
        "reviews": null,
        "weight": 123
    }

The **review** \- part of the response is now empty, but the rest of the reply remains intact! Look at the log of the `product-composite` services and you will find warnings:

    2015-04-02 15:13:36.344  INFO 29901 --- [eIntegration-1000] s.c.m.c.p.s.ProductCompositeIntegration  : GetRecommendations...
    2015-04-02 15:13:36.497  INFO 29901 --- [teIntegration-2] s.c.m.c.p.s.ProductCompositeIntegration  : GetReviews...
    2015-04-02 15:13:36.498  WARN 29901 --- [teIntegration-2] s.c.m.composite.product.service.Util     : Failed to resolve serviceId 'review'. Fallback to URL 'http://localhost:8081/review'.
    2015-04-02 15:13:36.500  WARN 29901 --- [teIntegration-2] s.c.m.c.p.s.ProductCompositeIntegration  : Using fallback method for review-service

I.e. the circuit breaker has detected a problem with the `review` service and routed the caller to the fallback method in the service consumer. In this case we simply return null but we could, for example, return data from a local cache to provide a best effort result when the `review` service is unavailable.

The circuit is still closed since the error is not that frequent:

![Hystrix][2500]

Let's increase the error frequency over the limits where Hystrix will open the circuit and start to fast fail (we use Hystrix default values to keep the blog post short…). We use the [Apache HTTP server benchmarking tool][2600] for this:

    ab -n 30 -c 5 localhost:8765/productcomposite/product/1

Now the circuit will be opened:

![Hystrix][2700]

…and subsequent calls will fast fail, i.e. the circuit breaker will redirect the caller directly to its fallback method without trying to get the reviews from the `review` service. The log will no longer contain a message that says `GetReviews...`:

    2015-04-02 15:14:03.930  INFO 29901 --- [teIntegration-5] s.c.m.c.p.s.ProductCompositeIntegration  : GetRecommendations...
    2015-04-02 15:14:03.984  WARN 29901 --- [ XNIO-2 task-62] s.c.m.c.p.s.ProductCompositeIntegration  : Using fallback method for review-service

However, from time to time it will let some calls pass through to see if they succeeds, i.e. to see if the `review` service is available again. We can see that by repeating the curl call a number of times and look in the log of the `product-composite` service:

    2015-04-02 15:17:33.587  INFO 29901 --- [eIntegration-1000] s.c.m.c.p.s.ProductCompositeIntegration  : GetRecommendations...
    2015-04-02 15:17:33.769  INFO 29901 --- [eIntegration-1000] s.c.m.c.p.s.ProductCompositeIntegration  : GetReviews...
    2015-04-02 15:17:33.769  WARN 29901 --- [eIntegration-1000] s.c.m.composite.product.service.Util     : Failed to resolve serviceId 'review'. Fallback to URL 'http://localhost:8081/review'.
    2015-04-02 15:17:33.770  WARN 29901 --- [eIntegration-1000] s.c.m.c.p.s.ProductCompositeIntegration  : Using fallback method for review-service
    2015-04-02 15:17:34.431  INFO 29901 --- [eIntegration-1000] s.c.m.c.p.s.ProductCompositeIntegration  : GetRecommendations...
    2015-04-02 15:17:34.569  WARN 29901 --- [ XNIO-2 task-1800] s.c.m.c.p.s.ProductCompositeIntegration  : Using fallback method for review-service
    2015-04-02 15:17:35.209  INFO 29901 --- [eIntegration-1000] s.c.m.c.p.s.ProductCompositeIntegration  : GetRecommendations...
    2015-04-02 15:17:35.402  WARN 29901 --- [ XNIO-2 task-2000] s.c.m.c.p.s.ProductCompositeIntegration  : Using fallback method for review-service
    2015-04-02 15:17:36.043  INFO 29901 --- [eIntegration-1000] s.c.m.c.p.s.ProductCompositeIntegration  : GetRecommendations...
    2015-04-02 15:17:36.192  WARN 29901 --- [ XNIO-2 task-21000] s.c.m.c.p.s.ProductCompositeIntegration  : Using fallback method for review-service
    2015-04-02 15:17:36.874  INFO 29901 --- [eIntegration-1000] s.c.m.c.p.s.ProductCompositeIntegration  : GetRecommendations...
    2015-04-02 15:17:37.031  WARN 29901 --- [ XNIO-2 task-220] s.c.m.c.p.s.ProductCompositeIntegration  : Using fallback method for review-service
    2015-04-02 15:17:41.148  INFO 29901 --- [eIntegration-1000] s.c.m.c.p.s.ProductCompositeIntegration  : GetRecommendations...
    2015-04-02 15:17:41.340  INFO 29901 --- [eIntegration-1000] s.c.m.c.p.s.ProductCompositeIntegration  : GetReviews...
    2015-04-02 15:17:41.340  WARN 29901 --- [eIntegration-1000] s.c.m.composite.product.service.Util     : Failed to resolve serviceId 'review'. Fallback to URL 'http://localhost:8081/review'.
    2015-04-02 15:17:41.341  WARN 29901 --- [eIntegration-1000] s.c.m.c.p.s.ProductCompositeIntegration  : Using fallback method for review-service

As we can see from the log output, every fifth call is allowed to try to call the `review` service (still without success…).

Let's start the `review` service again and try calling the composite service!

**Note:** You might need to be a bit patient here (max 1 min), both the service discovery server (Eureka) and the dynamic router (Ribbon) must be made aware of that a `review` service instance is available again before the call succeeds.

Now we can see that the response is ok, i.e. the review part is back in the response, and the circuit is closed again:

![Hystrix][2800]

## 7\. Summary

We have seen how _Netflix Hystrix_ can be used as a _circuit breaker_ to efficiently handle the problem with _chain of failures_, i.e. where a failing microservice potentially can cause a system outage of a large part of a microservice landscape due to propagating errors. Thanks to the annotations and starter dependencies available in Spring Cloud it is very easy to get started with Hystrix in a Spring environment. Finally the dashboard capabilities provided by Hystrix dashboard and Turbine makes it possible to monitor a large number of circuit breakers in a system landscape.

----

[Source](http://callistaenterprise.se/blogg/teknik/2015/06/08/building-microservices-part-4-dockerize-your-microservices/ "Permalink to Building Microservices, part 4. Dockerize your Microservices")

# Building Microservices, part 4. Dockerize your Microservices

###  08 June 2015  // [Magnus Larsson][222]

If you tried out the earlier blog posts in our [blog series][333] regarding building microservices I guess you are tired of starting the microservices manually at the command prompt? Let's dockerize our microservices, i.e. run them in Docker containers, to get rid of that problem!

I assume that you already heard of Docker and the container revolution? If not, there are tons of introductory material on the subject available on Internet, e.g. [Understanding Docker][444].

This blog post covers the following parts:

1. Install and configure Docker
2. Build Docker images automatically with Gradle
3. Configure the microservices for a Docker environment using Spring profiles
4. Securing access to the microservices using HTTPS
5. Starting and managing you Docker containers using Docker Compose
6. Test the dockerized microservices
    1. Happy days
    2. Scale up a microservice
    3. Handle problems
7. Summary
8. Next Step

Install Docker by following the [instructions for your platform][555].

&gt; We used [Boot2Docker v1.6.2 for Mac][666] when we wrote this blog post.

## 1.1 Configuration when using Boot2Docker

First, since we will start a lot of fine grained Java based microservices we will need some more memory (4GB) than the default value in the Linux virtual server that Boot2Docker creates for us.

The simplest way to do that is to stop Boot2Docker and recreate the virtual server with new memory parameters. If you upgraded Boot2Docker from an older version you should also perform a download of the virtual server to ensure that you have the latest version.

Execute the following commands to create a new virtual server with 4GB memory:

    $ boot2docker stop
    $ boot2docker download
    $ boot2docker delete
    $ boot2docker init -m 4096
    $ boot2docker info
    { ... "Memory":4096 ...}
    $ boot2docker start

The start command might ask you to set a number of environment variables like:

    export DOCKER_HOST=tcp://192.168.59.104:2376
    export DOCKER_CERT_PATH=/Users/magnus/.boot2docker/certs/boot2docker-vm
    export DOCKER_TLS_VERIFY=1

Add them to you preferred config-file, e.g. `~/.bash_profile` as in my case.

Now you can try Docker by asking it to start a CentOS based Linux server:

    $ docker run -it --rm centos

Docker will download the corresponding Docker image the first time it is used, so it takes some extra time depending on your Internet connection.

Just leave the server, for now, with an `exit` command:

    [root@fd5773461402 /]# exit

&gt; If you try to start a new CentOS server with the same command as above you will experience the magic startup time of a brand new server when using Docker, typically a sub-second startup time!!!

Secondly, add a line in your `/etc/hosts` \- file for easier access to the Docker machine. First get the IP address of the Linux server that runs all the Docker containers:

     $ boot2docker ip
    192.168.59.104

Add a line in your `/etc/hosts` file like:

    192.168.59.104 docker

Now you can, for example, `ping` your docker environment like:

    $ ping docker
    PING docker (192.168.59.103): 56 data bytes
    64 bytes from 192.168.59.103: icmp_seq=0 ttl=64 time=0.822 ms
    64 bytes from 192.168.59.103: icmp_seq=1 ttl=64 time=4.341 ms
    64 bytes from 192.168.59.103: icmp_seq=2 ttl=64 time=1.454 ms

We have extended the Gradle build files from the earlier [blog posts][333] to be able to automatically create Docker images for each microservice. We use a Gradle plugin developed by [Transmode][777].

As before we use Java SE 8 and Git so to get the source code perform:

    $ git clone https://github.com/callistaenterprise/blog-microservices
    $ cd blog-microservices
    $ git checkout -b B6 M6

The most important additions to the Gradle build-files are:

    buildscript {
        dependencies {
            classpath 'se.transmode.gradle:gradle-docker:1.2'

    ...

    apply plugin: 'docker'

    ...

    group = 'callista'
    mainClassName = 'se.callista.microservises.core.review.ReviewServiceApplication'

    ...

    distDocker {
        exposePort 8080
        setEnvironment 'JAVA_OPTS', '-Dspring.profiles.active=docker'
    }

    docker {
        maintainer = 'Magnus Larsson <magnus.larsson.ml@gmail.com>'
        baseImage = 'java:8'
    }

**Notes:**

1. First we declare a dependency to the `transmode-docker-plugin` and apply the `docker` \- plugin.

2. Next, we setup a variable, `group`, to give the Docker images a common group-name and another variable, `mainClassName`, to declare the main-class in the microservice.

3. Finally, we declare how the Docker image shall be built in the `distDocker` and `docker` declarations. See [plugin documentation][777] for details.

4. One detail worth some extra attention is the declaration of the `JAVA_OPTS` environment variable that we use to specify what Spring profile that the microservice shall use, see below for details.

We have also added the new task `distDocker` to the build commands in `build-all.sh`.

Execute the `build-all.sh` file and it will result in a set of Docker images like:

    $ ./build-all.sh
    ... lots of output ...

    $ docker images | grep callista
    callista/turbine                     latest              8ea25912aad7        43 hours ago        794.6 MB
    callista/monitor-dashboard           latest              f443c2cde704        43 hours ago        793 MB
    callista/edge-server                 latest              b32bb74788ac        43 hours ago        826.6 MB
    callista/discovery-server            latest              8eceaff6cc6b        43 hours ago        838.3 MB
    callista/auth-server                 latest              90041b13c564        43 hours ago        766.1 MB
    callista/product-api-service         latest              5081a18b9cac        44 hours ago        801.9 MB
    callista/product-composite-service   latest              c200820d6cdf        44 hours ago        800.6 MB
    callista/review-service              latest              1796c14c2a5a        44 hours ago        786.1 MB
    callista/recommendation-service      latest              4f4e490cb409        44 hours ago        786.1 MB
    callista/product-service             latest              5ed6a9620bce        44 hours ago        786.1 MB

To be able to run in a Docker environment we need to change our configuration a bit. To keep the Docker specific configuration separate from the rest we use a [Spring Profile][888] called `Docker` in our `application.yml`-files, e.g.:

    ---
    # For deployment in Docker containers
    spring:
      profiles: docker

    server:
      port: 8080

    eureka:
      instance:
        preferIpAddress: true
      client:
        serviceUrl:
          defaultZone: http://discovery:8761/eureka/

**Notes:**

1. Since all microservices will run in their own Docker container (e.g. in its own server with its own IP address) we don't need to care about port collisions. Therefore we can use a fixed port, `8080`, instead of dynamically assign ports as we have done until now.

2. Register our microservices to Eureka using hostnames in a Docker environment will not work, they will all get one and the same hostname. Instead we configure them to use its IP address during registration with Eureka.

3. The discovery service will be executing in a Docker container known under the name `discovery`, se below for details, so therefore we need to override the setting of the `serviceUrl`.

Docker runs the containers in a closed network. We will expose as few services outside of the internal Docker network as possible, e.g. the OAuth Authorization server and the Edge server. This means that our microservices will not be accessible directly from the outside. To protect the communication with the exposed services we will use HTTPS, i.e. use server side certificates, that will protect the OAuth communication from unwanted eavesdropping. The OAuth Authorization server and the Edge server uses a [self-signed certificate][999] that comes with the source code of this blog post.

&gt; Don't use this self-signed certificate for anything else than trying out our blog posts, it is not secure since its private part is publicly available in the source code!

Our API - microservice needs to communicate with the OAuth Authorization server to get information regarding the user (_resource owner_ in OAuth lingo). Therefore it needs to be able to act as a HTTPS client, validating the certificate that the OAuth Authorization service presents during the HTTPS communication. The API - microservice uses a trust store that comes with the source code of this blog post for that purpose.

For the OAuth Authorization server and the Edge server you can find the (not so) private certificate, `server.jks`, in the folder `src/main/resources` and the `application.xml/.yml` file in the same folder contains the SSL - configuration, e.g:

    server:
      ssl:
        key-store: classpath:server.jks
        key-store-password: password
        key-password: password

The API - microservice has its trust store, `truststore.jks`, and its configuration file in the folder `src/main/resources` as well. The SSL configuration looks a bit different since it only will act as a HTTPS-client:

    server:
      ssl:
        enabled: false
        # Problem with trust-store properties?
        #
        # Instead use: java -Djavax.net.debug=ssl -Djavax.net.ssl.trustStore=src/main/resources/truststore.jks -Djavax.net.ssl.trustStorePassword=password -jar build/libs/*.jar
        #
        # trust-store: classpath:truststore.jks
        trust-store: src/main/resources/truststore.jks
        trust-store-password: password

As you can see in the comment above we have experienced some problems with defining the trust store via the configuration file. Instead we use the `JAVA_OPTS` environment variable to specify it. If you look into the `build.gradle` \- file of the API - microservice you will find:

    distDocker {
        ...
        setEnvironment 'JAVA_OPTS', '-Dspring.profiles.active=docker -Djavax.net.ssl.trustStore=truststore.jks -Djavax.net.ssl.trustStorePassword=password'

To be able to start up and manage all our services with single commands we use [Docker Compose][101010].

&gt; We used [Docker Compose v1.2.0][1111] when we wrote this blog post.

With Docker Compose you can specify a number of containers and how they shall be executed, e.g. what Docker image to use, what ports to publish, what other Docker containers it requires to know about and so on…

For a simple container that don't need to know anything about other containers the following is sufficient in the configuration file, `docker-compose.yml`:

    rabbitmq:
      image: rabbitmq:3-management
      ports:
        - "5672:5672"
        - "15672:15672"

    discovery:
      image: callista/discovery-server
      ports:
        - "8761:8761"

    auth:
      image: callista/auth-server
      ports:
        - "9999:9999"

This will start up RabbitMQ, a discovery server and a OAuth Authorization server and publish the specified ports for external access.

To start up services that need to know about other containers we can use the `links` \- directive. e.g. for the API microservice:

    api:
      image: callista/product-api-service
      links:
        - auth
        - discovery
        - rabbitmq

This declaration will result in that the /etc/hosts file in the API container will be updated with one line per service that the API microservice depends on, e.g.:

    172.17.0.23	auth
    172.17.0.27	discovery
    172.17.0.25	rabbitmq

Ok, we now have all the new bits and pieces in place so we are ready to give it a try!

For an overview of the microservice landscape we are about to launch see previous blog posts, specifically [Part 111][1222] but also [Part 222][1333] and [Part 333][1444].

Start the microservice landscape with the following command:

    $ docker-compose up -d

**Note:** We have, a few times, noticed problems with downloading Docker images (unclear what triggers the problem). But after recreating the Boot2Docker virtual server as described in _1.1 Configuration when using Boot2Docker_ the download worked again.

This will start up all ten Docker containers. You can see its state withe the command:

    $ docker-compose ps
                Name                           Command               State                        Ports
    -------------------------------------------------------------------------------------------------------------------------
    blogmicroservices_api_1         /product-api-service/bin/p ...   Up      8080/tcp
    blogmicroservices_auth_1        /auth-server/bin/auth-server     Up      0.0.0.0:9999-&gt;9999/tcp
    blogmicroservices_composite_1   /product-composite-service ...   Up      8080/tcp
    blogmicroservices_discovery_1   /discovery-server/bin/disc ...   Up      0.0.0.0:8761-&gt;8761/tcp
    blogmicroservices_edge_1        /edge-server/bin/edge-server     Up      0.0.0.0:443-&gt;8765/tcp
    blogmicroservices_monitor_1     /monitor-dashboard/bin/mon ...   Up      0.0.0.0:7979-&gt;7979/tcp
    blogmicroservices_pro_1         /product-service/bin/produ ...   Up      8080/tcp
    blogmicroservices_rabbitmq_1    /docker-entrypoint.sh rabb ...   Up      0.0.0.0:15672-&gt;15672/tcp, 0.0.0.0:5672-&gt;5672/tcp
    blogmicroservices_rec_1         /recommendation-service/bi ...   Up      8080/tcp
    blogmicroservices_rev_1         /review-service/bin/review ...   Up      8080/tcp

You can monitor log output with the command:

    $ docker-compose logs
    ...
    rec_1       | 2015-06-01 14:20:55.295 cfbfc65f-8a5f-41cc-8710-51856105bf62 recommendation  INFO  XNIO-2 task-13 s.c.m.c.r.s.RecommendationService:53 - /recommendation called, processing time: 0
    rec_1       | 2015-06-01 14:20:55.296 cfbfc65f-8a5f-41cc-8710-51856105bf62 recommendation  INFO  XNIO-2 task-13 s.c.m.c.r.s.RecommendationService:62 - /recommendation response size: 3
    ...

…and as usual you can see the registered microservices in our discovery service, Eureka, using the URL `http://docker:8761`:

![system-landscape][1555]

## 6.1. Happy days

Let's try a happy days scenario, shall we?

First we need to access a OAuth Token using HTTPS (See [Part 333][1444] regarding details of the use of OAuth):

    $ curl https://acme:acmesecret@docker:9999/uaa/oauth/token
      -d grant_type=password -d client_id=acme
      -d username=user -d password=password -ks | jq .
    {
      "access_token": "d583cc8d-5431-4241-afbf-6c6e686899d8",
      "token_type": "bearer",
      "refresh_token": "cf3e2136-6fb3-4c23-b3ce-0d5118b5d538",
      "expires_in": 43199,
      "scope": "webshop"
    }

Store the Access Token in an environment variable as before:

    $ TOKEN=d583cc8d-5431-4241-afbf-6c6e686899d8

With the Access Token we can now access the API, again over HTTPS:

    $ curl -H "Authorization: Bearer $TOKEN"
      -ks 'https://docker/api/product/1046' | jq .
    {
      "productId": 1046,
      "name": "name",
      "weight": 123,
      "recommendations": [ ... ],
      "reviews": [ ... ]
    }

Great!

## 6.2. Scale up a microservice

Ok, let's spin up a second instance of one of the microservices. This can be done using the `docker-compose` `scale`-command:

    $ docker-compose scale rec=2

**Note:** `rec` is the name we gave the `recommendation` microservice in the docker-compose configuration file, `docker-compose.yml`.

This command will start up a second instance of the `recommendation` microservice:

    $ docker-compose ps rec
             Name                        Command               State    Ports
    ---------------------------------------------------------------------------
    blogmicroservices_rec_1   /recommendation-service/bi ...   Up      8080/tcp
    blogmicroservices_rec_2   /recommendation-service/bi ...   Up      8080/tcp

If you call the API several times with:

    $ curl -H "Authorization: Bearer $TOKEN"
      -ks 'https://docker/api/product/1046' | jq .

If you run the `docker-compose logs` command in a separate window, you will notice in the log-output, after a while, that the two `rec`-services take every second call.

    rec_2       | 2015-06-01 14:20:54.357 cb6ad766-c385-442b-afbe-de9222221a23 recommendation  INFO  XNIO-2 task-2 s.c.m.c.r.s.RecommendationService:53 - /recommendation called, processing time: 0
    rec_2       | 2015-06-01 14:20:54.358 cb6ad766-c385-442b-afbe-de9222221a23 recommendation  INFO  XNIO-2 task-2 s.c.m.c.r.s.RecommendationService:62 - /recommendation response size: 3

    ...

    rec_1       | 2015-06-01 14:20:55.295 cfbfc65f-8a5f-41cc-8710-51856105bf62 recommendation  INFO  XNIO-2 task-13 s.c.m.c.r.s.RecommendationService:53 - /recommendation called, processing time: 0
    rec_1       | 2015-06-01 14:20:55.296 cfbfc65f-8a5f-41cc-8710-51856105bf62 recommendation  INFO  XNIO-2 task-13 s.c.m.c.r.s.RecommendationService:62 - /recommendation response size: 3

Good, not let's cause some problems in the microservice landscape!

## 6.3. Handle problems

Let's wrap up the tests with introducing an error and see how our circuit breaker introduced in [Part 222][1333] acts in an Docker environment.

We have a backdoor in the review microservice that can be used to control its response times. If we increase the response time over the timeout configured in the circuit-breaker it will kick in…

The problem is that we can't access that backdoor from the outside (on the other hand, if the backdoor was accessible from the outside we would be in really big security problems…). To access the backdoor we need access to a sever that runs inside the closed Docker container network. Let's start one!

    $ docker run -it --rm --link blogmicroservices_rev_1:rev centos
    [root@bbd3e4154803 /]#

That wasn't that hard, was it?

Ok, now we can use the backdoor to set the response time of the review microservice to 10 secs:

    [root@bbd3e4154803 /]# curl "http://rev:8080/set-processing-time?minMs=10000&amp;maxMs=10000"

Exit the container and retry the call to the API. It will respond very slowly (3 sec) and will not contain any review information:

    [root@bbd3e4154803 /]# exit

    $ curl -H "Authorization: Bearer $TOKEN" -ks
      'https://docker/api/product/1046' | jq .
    {
      "productId": 1046,
      "name": "name",
      "weight": 123,
      "recommendations": [ ... ],
      "reviews": null
    }

If you retry the call once more you will again see a very long response time, i.e. the circuit breaker has not opened the circuit yet. But if you perform two requests directly after each other the circuit will be opened.

&gt; We have configured the circuit breaker to be very sensitive, just for demo purposes.

This can be seen in the circuit breakers dashboard like:

![system-landscape][1666]

**Note:** URL to the circuit breaker: <http: docker:7979="" hystrix="" monitor?stream="http%3A%2F%2Fcomposite%3A8080%2Fhystrix.stream">

If you retry a call you will see that you get an **immediate response**, still without any review information of course, i.e. the circuit breaker is now open.

Let's heal the broken service! If you start a new container and use the backdoor to reset the response time of the review microservice to 0 sec and then retry the call to the API everything should be ok again, i.e. the system is self-healing!

    $ docker run -it --rm --link blogmicroservices_rev_1:rev centos
    [root@bbd3e4154803 /]# curl "http://rev:8080/set-processing-time?minMs=0&amp;maxMs=0"
    [root@bbd3e4154803 /]# exit

    $ curl -H "Authorization: Bearer $TOKEN"
      -ks 'https://docker/api/product/1046' | jq .
    {
      "productId": 1046,
      "name": "name",
      "weight": 123,
      "recommendations": [ ... ],
      "reviews": [ ... ]
    }

**Note:** The circuit breaker is configured to probe the open circuit after 30 seconds to see if the service is available again, i.e. to see if the problem is gone so it can close the circuit again. Probing is done by letting one of the incoming request through, even though that the circuit actually is open.

We have seen how we with very little effort can dockerize our microservices and run our microservices as Docker containers. Gradle can help us to automatically build Docker images, Spring profiles can help us to keep Docker specific configuration separate from other configuration. Finally Docker Compose makes it possible to start and manage all microservices, used for example by an application, with a single command.

We have already promised to demonstrate how the ELK stack can be used to provide centralized log management of our microservices. Before we demonstrate that we however need to consider how to correlate log event written by various microservices to its own log-files. So that will be the target for the next blog post in the [Blog Series - Building Microservices][1777], stay tuned…


[1]: http://callistaenterprise.se/assets/medarbetare/magnuslarsson_mini.png
[2]: /om/medarbetare/magnuslarsson/
[3]: /blogg/teknik/2015/03/25/an-operations-model-for-microservices/
[4]: http://projects.spring.io/spring-cloud/
[5]: http://netflix.github.io
[6]: https://spring.io/blog/2015/03/04/spring-cloud-1-0-0-available-now
[7]: http://spring.io/projects
[8]: /blogg/teknik/2014/04/15/a-first-look-at-spring-boot/
[9]: (/blogg/teknik/2015/03/25/an-operations-model-for-microservices/)
[10]: http://callistaenterprise.se/assets/blogg/build-microservices-part-1/mapping-table.png
[11]: http://ghostbusters.wikia.com/wiki/Zuul
[12]: http://www.programmableweb.com/apis/directory
[13]: http://callistaenterprise.se/assets/blogg/build-microservices-part-1/system-landscape.png
[14]: http://callistaenterprise.se/assets/blogg/build-microservices-part-1/source-code.png
[15]: /blogg/teknik/2014/04/14/a-first-look-at-gradle/
[16]: http://undertow.io
[17]: http://docs.spring.io/spring/docs/current/spring-framework-reference/html/mvc.html
[18]: https://spring.io/blog/2009/03/27/rest-in-spring-3-resttemplate
[19]: /blogg/teknik/2014/04/22/c10k-developing-non-blocking-rest-services-with-spring-mvc/
[20]: https://github.com/callistaenterprise/blog-microservices/blob/B1/microservices/core/product-service/build.gradle
[21]: https://github.com/callistaenterprise/blog-microservices/blob/B1/microservices/support/discovery-server/build.gradle
[22]: https://github.com/callistaenterprise/blog-microservices/blob/B1/microservices/support/discovery-server/src/main/java/se/callista/microservises/support/discovery/EurekaApplication.java
[23]: https://github.com/callistaenterprise/blog-microservices/blob/B1/microservices/support/edge-server/src/main/java/se/callista/microservises/support/edge/ZuulApplication.java
[24]: https://github.com/callistaenterprise/blog-microservices/blob/B1/microservices/support/edge-server/src/main/resources/application.yml
[25]: https://github.com/callistaenterprise/blog-microservices/blob/B1/microservices/core/product-service/src/main/java/se/callista/microservises/core/product/ProductServiceApplication.java
[26]: https://github.com/callistaenterprise/blog-microservices/blob/B1/microservices/composite/product-composite-service/src/main/java/se/callista/microservices/composite/product/service/Util.java
[27]: https://github.com/callistaenterprise/blog-microservices/blob/B1/microservices/composite/product-composite-service/src/main/java/se/callista/microservices/composite/product/service/ProductCompositeIntegration.java
[28]: http://curl.haxx.se
[29]: http://stedolan.github.io/jq/
[30]: http://callistaenterprise.se/assets/blogg/build-microservices-part-1/eureka.png
[31]: http://callistaenterprise.se/assets/blogg/build-microservices-part-1/eureka-2-review-instances.png
[32]: http://callistaenterprise.se/assets/blogg/build-microservices-part-1/log-from-2-review.instances.png
[33]: /blogg/teknik/2015/05/20/blog-series-building-microservices/

[100]: http://callistaenterprise.se/assets/medarbetare/magnuslarsson_mini.png
[200]: /om/medarbetare/magnuslarsson/
[300]: /blogg/teknik/2015/04/10/building-microservices-with-spring-cloud-and-netflix-oss-part-1/
[400]: http://projects.spring.io/spring-cloud/
[500]: http://netflix.github.io
[600]: /blogg/teknik/2015/03/25/an-operations-model-for-microservices/
[700]: https://pragprog.com/book/mnee/release-it
[800]: http://martinfowler.com/bliki/CircuitBreaker.html
[900]: http://callistaenterprise.se/assets/blogg/build-microservices-part-2/circuit-breaker.png
[1000]: http://callistaenterprise.se/assets/blogg/build-microservices-part-1/mapping-table.png
[1100]: http://callistaenterprise.se/assets/blogg/build-microservices-part-2/hystrix-sample.png
[1200]: http://callistaenterprise.se/assets/blogg/build-microservices-part-2/system-landscape.png
[1300]: http://callistaenterprise.se/assets/blogg/build-microservices-part-2/source-code.png
[1400]: https://www.rabbitmq.com
[1500]: https://github.com/callistaenterprise/blog-microservices/blob/B2/microservices/composite/product-composite-service/build.gradle
[1600]: https://github.com/callistaenterprise/blog-microservices/blob/B2/microservices/support/turbine/build.gradle
[1700]: https://github.com/callistaenterprise/blog-microservices/blob/B2/microservices/support/turbine/src/main/java/se/callista/microservises/support/turbine/TurbineApplication.java
[1800]: https://github.com/callistaenterprise/blog-microservices/blob/B2/microservices/support/monitor-dashboard/src/main/java/se/callista/microservises/support/monitordashboard/HystrixDashboardApplication.java
[1900]: https://github.com/callistaenterprise/blog-microservices/blob/B2/microservices/composite/product-composite-service/src/main/java/se/callista/microservices/composite/product/service/ProductCompositeIntegration.java
[2000]: http://curl.haxx.se
[21000]: http://stedolan.github.io/jq/
[220]: https://www.rabbitmq.com/download.html
[2300]: http://callistaenterprise.se/assets/blogg/build-microservices-part-1/eureka.png
[2400]: http://callistaenterprise.se/assets/blogg/build-microservices-part-2/hystrix.png
[2500]: http://callistaenterprise.se/assets/blogg/build-microservices-part-2/circuit-closed-on-minor-error.png
[2600]: http://httpd.apache.org/docs/2.4/programs/ab.html
[2700]: http://callistaenterprise.se/assets/blogg/build-microservices-part-2/circuit-open-on-major-error.png
[2800]: http://callistaenterprise.se/assets/blogg/build-microservices-part-2/circuit-closed-again.png
[2900]: /blogg/teknik/2015/05/20/blog-series-building-microservices/

[111]: http://callistaenterprise.se/assets/medarbetare/magnuslarsson_mini.png
[222]: /om/medarbetare/magnuslarsson/
[333]: http://callistaenterprise.se/blogg/teknik/2015/05/20/blog-series-building-microservices/
[444]: https://docs.docker.com/introduction/understanding-docker/
[555]: http://docs.docker.com/installation/
[666]: http://docs.docker.com/installation/mac/
[777]: https://github.com/Transmode/gradle-docker
[888]: http://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-profiles.html
[999]: http://en.wikipedia.org/wiki/Self-signed_certificate
[101010]: https://docs.docker.com/compose/
[1111]: http://docs.docker.com/compose/install/
[1222]: /blogg/teknik/2015/04/10/building-microservices-with-spring-cloud-and-netflix-oss-part-1/
[1333]: /blogg/teknik/2015/04/15/building-microservices-with-spring-cloud-and-netflix-oss-part-2/
[1444]: /blogg/teknik/2015/04/27/building-microservices-part-3,%20secure%20API's%20with%20OAuth/
[1555]: http://callistaenterprise.se/assets/blogg/build-microservices-part-4/docker-eureka.png
[1666]: http://callistaenterprise.se/assets/blogg/build-microservices-part-4/docker-hystrix.png
[1777]: /blogg/teknik/2015/05/20/blog-series-building-microservices/


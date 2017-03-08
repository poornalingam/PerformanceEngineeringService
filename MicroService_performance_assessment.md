# How and What will be analyzed
- [Performance Metrics for all software and physical stack](/Performance-Metric.md)
- [Tools for performance analysis](/Performance_Analysis_Tools.md)
- [K8S and Container level performance analysis](/K8S-Container-performance-monitoring.md)
- [MicroServices performance analysis](/MicroService_performance_assessment.md)
- [Techniques for performance analysis](/Method-PerformanceAnalysis.md)
- Scripting language specific performance analysis

# MicroServices Performance Assessment

 **_Need_**: A look at a diagram must say what is taking time explicitly.

One thing that can help is to be able to inspect individual transactions and see what is going on. The image below shows Google’s Cloud Trace in action, showing how the 106ms adds up for the /add_point endpoint. Basically, Cloud Trace provides distributed stack traces.

It is available only in the Google Cloud for RPCs (Remote Procedure Call). We can use in GCP environment. Yet to find a tool for AWS and private cloud.
![Google Cloud](/Images/Google-Cloud-for-RPC.png)

### Google Dapper

It’s a google white paper to build a tool to track microservices and its latency.
![Google Dapper](/Images/MicroService_Monitoring.jpg)

### Monitoring requirements:
1. Track the number of connections from each client applications. Its to find rogue applications which sends lots of requests
2. Monitor the connection pool utilization for each external Microservices
3. Log the circuit breakers calls and alert the failure
4. Monitor the message queue between pub and sub MicroService call
5. Capability to monitor the latency between Microservices calls

Resource|Type| Metrics
---|---|---
TBD||

### Performance Patterns in Microservices-Based Integrations
Design patterns can ensure good architectural design, but these alone are not enough to address performance challenges. This is where performance patterns come into play. When implemented correctly, these can really help build a scalable solution.

  1. **Throttling** - is one technique that can be used to prevent any misbehaving or rogue application from overloading or bringing down our application by sending more requests than what our application can handle.
    - Throttle the connections for each client depends on number of connections expectations
  2. **Dedicated thread pool/Bulkhead** - Consider,  you need to connect to five different microservices using REST over HTTP. You are also using a library to use a common thread pool for maintaining these connections. If, for some reason, one of the five services starts responding slowly, then all your pool members will be exhausted waiting for the response from this service.
    - To minimize the impact, it is always a good practice to have a dedicated pool for each individual service.

    ![microservices Connection Pool](/Images/MicroService-ConnectionPool.png)

  3. A **Circuit Breaker** is a design pattern, which is used to minimize the impact of any of the downstream being not accessible or down (due to planned or unplanned outages). Circuit breakers are used to check the availability of external systems/services, and in case these are down, applications can be prevented from sending requests to these external systems.

  4. **Asynchronous integration** - _Most performance issues related to integrations can be avoided by decoupling the communications between microservices._ The asynchronous integration approach provides one such mechanism to achieve this decoupling. Any standard message broker system can be used to provide publish-subscribe capabilities.

    _Decoupling between producers and receivers/subscribers is achieved with the use of a message broker_
    ![MicroServices Message   Broker](/Images/MicroServices-MessageBroker.png)

[Back to Top](#how-and-what-will-be-analyzed)

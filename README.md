# refarch-cloudnative-wfd-devops
DevOps repository for the cloud native reference architecture for the microservices workstream

[![Create wfd Deployment Toolchain](https://new-console.ng.bluemix.net/devops/graphics/create_toolchain_button.png)](https://new-console.ng.bluemix.net/devops/setup/deploy/?repository=https%3A//github.com/jesusmah/test-devops)



# refarch-cloudnative-wfd-devops-cf

This repository contains the DevOps toolchain for managing and deploying the microservices making up the What's For Dinner app to Bluemix as Cloud Foundry apps.

# How to use

1. The "What is for dinner Toolchain" creation view will open.
2. In this window, you will be asked to specify the following properties for your toolchain:
 1. Specify a name for your toolchain that is unique among all toolchains on your Bluemix namespace DevOps section.
 2. Click on the "GitHub" icon. This opens the GitHub settings. Please, give your desired name to each of the repos that will be cloned.
 3. Click on the "Delivery Pipeline" icon. This opens the delivery pipeline settings:
   * Specify the Bluemix domain for your apps' routes (by default: mybluemix.net).
    * Specify the build branch you would like your delivery pipelines to build the code from (by default: master).
     * Specify a unique identifier which must make your apps' names *unique in Bluemix public* (by default: toolchain's creation timestamp).
3. Click the Create button to complete the toolchain creation.
4. After creating the toolchain, make sure to deploy the microservices in the following order:
 1. The Eureka server. To do this, execute the wfd-eureka-cf-ad delivery pipeline
 2. The Config server, by running the wfd-config-cf-ad delivery pipeline
 3. All other microservices, by executing their delivery pipelines

# Details

This toolchain contains a github clone tool, and a delivery pipeline for each of the microservices.
The github clone tool creates a cloned repository for the microservice.
Each delivery pipeline consists of two stages:

1. Build. This stage has one job, which runs a gradle build of the microservice.
2. Delivery Pipeline. This stage has 4 jobs:
 1. **Deploy CF App**. This job deploys a new version of the microservice.
 2. **Active Deploy - Begin**. This job creates an active deployment for the microservice, and advances this to the Test phase.
 3. **Test New Version**. This job is empty by default. It can be populated with any required tests.
 4. **Active Deploy - Complete**. This job continues the active deployment for the microservice, and completes it.

# Considerations
## Order of deployment
All microservices depend on Eureka for service registration and discovery. This is established by creating a service associated to the Eureka server that can be bound to the other microservices. This service is created the very first time the Eureka server is deployed, and it must exist before other services are deployed so they can bind it to them.

Dynamic configuration is implemented using the Config server. The Config server is accessed by microservices by binding them to the service that is associated with the Config server. This service is created on the first deployment of the Config server, and therefor is required to exist before any of the other microservices are deployed.

## Eureka & active deploy

Active deploy switches from an old to a new version of an app by switching the route from an old version of an app to the new version of that app. Oftentimes this is fine, but in the case of Eureka there is a complication. Eureka keeps an in-memory database of all app that have registered with it. Any new version of the Eureka app will not immediately have the registrations the current version has, and until it does, its service cannot fully replace the service of the old version. This means that there will be some period of time in which Eureka's service will be degraded.

Microservices start registering with Eureka as soon as they can access the Eureka server, which in the context of Active Deploy is the end of the rampup phase. In order to make the process of registering with a new Eureka as fast as possible, we make the rampup phase as short as possible: one second (1s). In doing this, the period of time in which the Eureka service is degraded is  minimized.

By doing this, we have observed it to take anywhere from 50-80 seconds until the new Eureka contains the registrations of all microservices, and Eureka's service is fully restored.

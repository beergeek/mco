# MCollective

## Introduction

MCollective is a powerful ad hoc management tool that is included with Puppet Enterprise.

## Terminology

### Server

The MCollective Server is the service that resides on the nodes.  The MCollective Server connects to the middleware broker, determines if requests are valid, actions the requests and formats the replies.  This service has numerous plugins, including connectors, security, authorisation, data, registration and agents.  The SSL plugins is used by MCollective as a security mechanism for credentials, unique tokens, plus protection for message TTL and Message Time property.  The actual message payload is not encrypted with the SSL plugin, but is handled at the middleware layer. The server will connect to the middleware broker on initial start sending registration information and then provide a heartbeat on a regular basis.  The server will remain connected to the middleware broker as long as the services is running and no network systems terminate this connection.  Reconnect will occur if the connect is dropped; this is done periodically.

### Client

Like MCollective Servers, the Clients connect to the middleware broker, but only for the duration of the messaging process.  Clients use plugins similar to Servers for connecting and security purposes.  Other plugins allow the Clients to discover Servers that meet certain criteria, such as the presence of particular agents or certain Facter facts.

### Middleware

Middleware is the layer that provides the transport mechanism for data flow in the environment.  The middleware broker is the service that all MCollective servers and clients connect to.  The middleware broker received messages from the clients and forwards them to the appropriate servers; this can be via broadcast or point-to-point.  The broker also handles the replies from the servers to the clients.  The Middleware provides end-to-end encryption of the payload via TLS.

The Middleware Broker has numerous queues and topics that are used to route certain messages.

For Puppet Enterprise the Middleware Broker is Apache ActiveMQ.

### Agent

A MCollective Agent is a piece of Ruby code that lives on the MCollective Servers.  Each Agent has an associated Data Definition Language (ddl) file used to describe the actions, plus validate inputs and outputs of the Agents.  An example of an Agent would be a Service agent used to start, stop and provide status of services on a node.

### Action

Each MCollective Agent will have one or more actions.  These actions are the actual work to be performed, such as starting a service, stopping a service or getting the status of a service.

### Application

A MCollective Applications are the single executables for MCollective Clients.  They work with the DDL files as do the Agents.  The Application is the executable that will address the Agents on the nodes and provide the required Action and inputs, whilst receiving and formatting the responses.

## Collective and Subcollectives

These are namespaces in the MCollective environment that Servers and Clients can subscribed to.  Collectives and subcollectives can be used as logical, geogrphical or security boundaries.  Both Clients and Servers can belong to one and only one collective, but can belong to zero or more subcollectives.

### Topics

Topics implement the publish and subscribe semantics, they have a zero-to-many subscribes.  Therefore messages are published to zero or many subscribers.  In MCollective the actual collective and subcollectives are topics.

### Queues

Queues implement the load balancer semantics.  There is a queue for each collective and agent combination, e.g. mcollective_package.

## Dataflow

The Server will connect to the Middleware Broker and wait for requests; heartbeat messages are sent to assist in keeping the connection open.

When a Client executes an application, the Client will connect to the Middleware Broker and send a discovery message to determine if any servers are valid (unless the application is targeted to specific nodes). The Client will create a temporary queue that the responding nodes will send replies to.  The Client receives the replies and formats a targeted message to the subset of Servers that have replied.  The Client sends the message to the individual Servers and once again creates a temporary queue for the nodes to reply to.  The Server performs the task requested and responds appropriately to the temporary queue, which the Client will format for the user to review.  The Client will timeout temporary queues after a specified period.

## Use Cases

MCollective is a very powerful tool that can be used in many ah hoc situations, such as:

* Vulnerability checking
* Updating of application code base
* Audit and compliance checking
* Running Puppet on numerous nodes

The Puppet Enterprise version of MCollective includes several Agents and Applications that can be used for numerous purposes and situations.

### Vulnerability checking

Puppet Enterprise includes the mcollective-package-agent Agent and Application.  This MCollective agent utilises the Resource Abstraction Layer of Puppet to perform ad hoc management of packages on various operating systems.  In 2014 there was the [Heartbleed vulnerability]('https://en.wikipedia.org/wiki/Heartbleed') which took companies months to fix.  With MCollective the vulnerable nodes could be determined in realtime, even if Puppet was not managing the particular resource.

```shell
peadmin@master:~$ mco package openssl status

 * [ ============================================================> ] 4 / 4

   database0.pdx.puppet.vm: openssl-1.0.1e-42.el6_7.4.x86_64
         lb0.pdx.puppet.vm: openssl-1.0.1e-42.el6_7.4.x86_64
      master.inf.puppet.vm: openssl-1.0.1e-42.el6_7.4.x86_64
       repo0.pdx.puppet.vm: openssl-1.0.1e-42.el6_7.4.x86_64

Summary of Arch:

   x86_64 = 4

Summary of Ensure:

   1.0.1e-42.el6_7.4 = 4


Finished processing 4 / 4 hosts in 655.76 ms
```

Puppet could be used to then rememdy this issue, or this could be done via MCollective as well. This example uses filtering to only update Linux nodes (`-F kernel=Linux`) which is determined by Facter facts:

```shell
peadmin@master:~$ mco package openssl update -F kernel=Linux

 * [ ============================================================> ] 4 / 4


Summary of Ensure:

   1.0.1e-48.el6_8.1 = 4


Finished processing 4 / 4 hosts in 55788.95 ms
```

In this case all four instances of Openssl were updated using MCollective. This can be confirmed again via:

```shell
peadmin@master:~$ mco package openssl status

 * [ ============================================================> ] 4 / 4

   database0.pdx.puppet.vm: openssl-1.0.1e-48.el6_8.1.x86_64
         lb0.pdx.puppet.vm: openssl-1.0.1e-48.el6_8.1.x86_64
      master.inf.puppet.vm: openssl-1.0.1e-48.el6_8.1.x86_64
       repo0.pdx.puppet.vm: openssl-1.0.1e-48.el6_8.1.x86_64

Summary of Arch:

   x86_64 = 4

Summary of Ensure:

   1.0.1e-48.el6_8.1 = 4


Finished processing 4 / 4 hosts in 122.19 ms
```

Noting that many nodes may be affected by this command, MCollective can perform requests in batches (`--batch`), with a user-determined period between each batch (`--batch-sleep`). In this example the batch size is two nodes, with a sleep of 10 seconds between each batch execution:

```shell
peadmin@master:~$ mco package openssl status --batch 2 --batch-sleep 10

 * [ ============================================================> ] 4 / 4
   database0.pdx.puppet.vm: openssl-1.0.1e-48.el6_8.1.x86_64
         lb0.pdx.puppet.vm: openssl-1.0.1e-48.el6_8.1.x86_64
       repo0.pdx.puppet.vm: openssl-1.0.1e-48.el6_8.1.x86_64
      master.inf.puppet.vm: openssl-1.0.1e-48.el6_8.1.x86_64

Summary of Arch:

   x86_64 = 4

Summary of Ensure:

   1.0.1e-48.el6_8.1 = 4


Finished processing 4 / 4 hosts in 20262.48 ms
```

As can be seen MCollective can be quickly used to determine the vulnerability of your environment to certain issues and then Puppet or MCollective can be used to resolve those issues.


## Updating Application Code Base

Many people need to update the code base for applications at a point in time or as part of a schedule.  This can include updating code in Tomcat, JBoss or Weblogic.  Many companies need to change the code at a specific time and not spread over 30 minutes, which Puppet would do if run in the traditional method.

MCollective can be used to update application code base on countless nodes at the same time.  A MCollective agent and application can be created that utilises Puppet to perform this task.  The task may look something like:

1. Disable Puppet
2. Stop a service (e.g. JBoss)
3. Enable a repository configuration (e.g. a configuration for YUM)
4. Upgrade a package
5. Disable a repository configuration
6. Start a service
7. Enable Puppet

Puppet is disabled so Puppet does not restart a service that is stopped during update. The repository configuration may not require enabling, but it is good to check.

Having the code base as a package is smart as the package can be version and Puppet can use these versions to get the right version of the package.

An example of this can be seen in this [example](https://github.com/beergeek/beergeek-app_update/blob/yum_demo/files/app_update.rb). The `files` directory of this module contains all the MCollective code.


## Audit and Compliance checking

MCollective can be used to audit nodes to ensure machines are compliant.  This can be performed using the MCollective Agents and Applications that are part of Puppet Enterprise.  These Agents use the Puppet Resource Abstraction Layer (RAL) to perform the actions.

To check all users on all nodes the following can be performed:

```shell
mco rpc puppetral search type=user
```

Many Puppet resource types can be audit using this procedure, such as `user`, `group`, `package`, `service`, `mysql_database` etc.

Resource types that do not have `self.instances` implemented, such as the `file` resource, individual resources can be audited:

```shell
mco rpc puppetral find type=file title=/etc
```

Each of the above commands will return a hash of resource(s) for each node interrogated.


## Running Puppet on multiple nodes

MCollective can be used to run Puppet on multiple nodes at once.  A MCollective client can be setup on services such as Bamboo or Jenkins to run Puppet when code changes occur.

The following example runs Puppet on nodes that have been classified with the class of `apache`.

```shell
peadmin@master:~$ mco puppet runonce -C apache

 * [ ============================================================> ] 3 / 3




Finished processing 3 / 3 hosts in 214.11 ms
```

Many companies use MCollective as part of their build pipeline to facilitate the update of code or to run tasks.  MCollective is a very flexible tool that can perform a multitude of tasks.

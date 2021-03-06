OVERVIEW
--------

This example is almost identical to the service1 example with the difference being that the service does
not manage its transaction participants itself and instead delegates to a java "integration API".

Normally when a services wishes to join an existing transaction it sends a message to the coordinator
containing HTTP endpoints on which it will be notified when the transaction progresses through its
prepare and commit stages. The integration API simplifies this task and instead the service enlists a
participant which implements a callback interface. The API will then invoke the callback transparently
(to the service). This makes managing participants much cleaner for the service writer.

The changes to the service1 quickstart are:

1. The pom includes two extra dependencies (restat-integration and narayana-jts-jacorb).

2. The integration API is implemented as a JAX-RS service so it needs to be
   registered with the container running the example service. The example runs in th
   `e Grizzly JAX-RS
   container and the registration is done by adding the package name of the "ParticipantResource"
   service (see quickstart.JaxrsServer for details) in the src folder.
   The integration API also needs to be told what URL the web service container is running on. This is done
   during container startup (see quickstart.JaxrsServer for details).

3. Modify the service to use the new API by registering a participant whenever it receives transactional
   service requests (quickstart.TransactionAwareResource#someServiceRequest()). This includes
   - registering a Participant for each service request


USAGE
-----

Prior to running the example make sure that the [RESTAT coordinator is deployed](../../README.md#usage).

    mvn clean compile exec:exec

or use the run.sh or run.bat script.


EXPECTED OUTPUT
---------------

[The examples run under the control of maven so you will need to filter maven output from example output.]

1. You will see lines of output showing the the two JAX-RS services being registered:

        INFO: Root resource classes found:
          class org.jboss.narayana.rest.integration.ParticipantResource
          class quickstart.TransactionAwareResource

2. A message showing the client committing transaction:

    Client: Committing transaction

3. A prepare and commit message is generated by each service work unit. This is repeated for both
work units:

        Service: preparing: wId=2
        Service: preparing: wId=1
        Service: committing: wId=1
        Service: committing: wId=2

4. The client checks that the service got both commit requests:

        SUCCESS: Both service work loads received commit requests


WHAT JUST HAPPENED?
-------------------
1. We deployed a JAX-RS servlet that implements a RESTful interface to the Narayana transaction manager (TM)
(running in an AS7 or AS6 container).

2. The client (MultipleParticpants.java) started an embedded web server (JaxrsServer.java) for hosting web services.

3. The client then started a REST Atomic Transaction and got back two urls: one for completing the transaction
and one for use with enlisting durable participants (implemented by the class TransactionAwareResource.java)
into the transaction.

4. The client then made two HTTP requests to a web service and passed the participant enlistment url as part
of the context of the request (using a query parameter).

5. The participants used the enlistment url to join the transaction. In this naive example we assumes that
each request is for a separate unit of transactional work and in this way we end up with multiple participants
being involved in the transaction. Having more than one participant means we can demonstrate that either all
participants will commit or none them will.

6. The client commits the transaction using the resource url it got back when it created the transaction.

7. The transaction manager implementation knows that there are two resources involved and asks them both to
prepare their work (using the resource urls it got from the participants when they enlisted into the transaction). The integration API receives these requests and calls the service particpant prepare callback.
If the participant resources respond with the correct Vote the transaction manager
proceeds to request commitment. Refer to the REST Atomic Transactions spec for full details.

8. The client sends a request to the service asking how many times it was asked to commit work units.
If it was twice then SUCCESS is printed otherwise FAILURE is printed.


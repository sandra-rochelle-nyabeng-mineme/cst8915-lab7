# Lab 7 – CST8915 Full-stack Cloud-native Development  

## Sandra Rochelle Nyabeng Mineme - 041268641



 **YouTube Link:** *https://youtu.be/KbqXqCk7UVA*


# RabbitMQ Configuration Analysis

## Is RabbitMQ Stateless or Stateful?

RabbitMQ is a **stateful** application.

### Why?
- RabbitMQ stores:
  - Queues
  - Messages
  - Exchanges and bindings
  - User credentials and permissions
- All of this data must persist **beyond the lifecycle of the container**.

A stateless application can be deleted and recreated without losing data.  


## **2. What Are the Implications of Running RabbitMQ Without Persistent Storage?**

In the `algonquin-pet-store-all-in-one.yaml` file, RabbitMQ is deployed **without a PersistentVolume (PV)** or **PersistentVolumeClaim (PVC)**.  
This causes serious problems:

**Data loss**  
All queues and messages are erased if the Pod crashes, restarts, or is rescheduled.

**Loss of application state**  
Exchanges, bindings, and user configurations disappear.

**Unreliable messaging**  
Microservices like the Order Service and Product Service cannot depend on RabbitMQ when messages are not durable.

**Unstable network identity**  
RabbitMQ is deployed as a **Deployment**, not a StatefulSet.  
This means:
- Changing pod names  
- Changing DNS endpoints  
- Clients lose connection after each restart  

 **Breaks the Algonquin Pet Store workflow**  
Without persistent message storage, orders and product, operations can fail or disappear.


## **3. What Happens When the RabbitMQ Pod Is Deleted or Restarted?**

When the RabbitMQ Pod is deleted, restarted, or rescheduled:

| Event | Result |
|-------|--------|
| Pod restart | All in-memory messages are lost |
| Pod deletion | RabbitMQ starts empty, with no queues or messages |
| Pod rescheduled to another node | Full data loss and new network identity |
| Microservices reconnect | They fail until RabbitMQ is fully recreated |

It means:
The entire message state of the system is erased.  
The application becomes unreliable and inconsistent.


## **4. Potential Solutions to Fix the RabbitMQ Configuration**

Below are **research-based**, industry-standard solutions:


### **Solution A: Use a StatefulSet**
StatefulSet provides:
- Stable pod names  
- Stable network identity  
- Correct ordering (for clustered RabbitMQ)  

A StatefulSet ensures each pod has a persistent identity, unique network name, and stable storage, which is crucial for applications like databases that need to maintain state

### **Solution B: Add Persistent Storage**
Use **PersistentVolume (PV)** + **PersistentVolumeClaim (PVC)**:
A StatefulSet in Kubernetes does not automatically come with persistent storage by default. However, it can be configured to use persistent storage through PersistentVolumeClaims (PVCs).

```yaml
volumeMounts:
  - mountPath: /var/lib/rabbitmq
    name: rmqdata

volumes:
  - name: rmqdata
    persistentVolumeClaim:
      claimName: rabbitmq-pvc 
```
**Benefits**:

Messages survive pod restarts

- Queues and bindings are preserved
- The broker becomes reliable and consistent
- This prevents the complete data loss currently present in the lab.


### Solution C: Use an Operator (RabbitMQ Cluster Operator)
The RabbitMQ Cluster Operator is a Kubernetes Operator that automates the management and operation of RabbitMQ clusters in Kubernetes environments. It simplifies the deployment, scaling, and management of RabbitMQ clusters, making it easier to handle the lifecycle of RabbitMQ instances in a Kubernetes environment.
The RabbitMQ operator (official) automates:

- Cluster creation
- Scaling
- Resiliency
- Monitoring
- TLS configuration

It also ensures correct data persistence and networking.

### Other solutions: Use a Fully Managed Service Instead (Recommended in Cloud Environments)

Instead of self-hosting RabbitMQ, most companies use a managed message broker such as:

- Azure Service Bus
- Amazon SQS/SNS
- Google Pub/Sub

Because they completely eliminate the operational challenges of running RabbitMQ.

##  **5. Does Using Azure Service Bus Solve the Issues Identified With RabbitMQ Configuration in This Lab?**

Yes. Azure Service Bus solves all the issues.

### Why?
### 1. Persistence is built-in

Messages are automatically stored durably—no PVC, PV, StatefulSet, or disk required.

### 2. High Availability is automatic

RabbitMQ requires clustering, volume replication, and multiple pods.
Azure Service Bus gives HA out-of-the-box.

### 3. No pod restarts or Kubernetes failures

Since there is no container:

No restarts
No rescheduling
No node failure
No data loss

### 4. No maintenance

No need to manage disks, volumes, users, or configuration.

### 5. Stronger guarantees

Azure Service Bus offers:

- Exactly-once or at-least-once delivery
- Dead-letter queue
- Automatic retries
- Enforced message ordering
All of which are difficult and complex to set up with RabbitMQ on Kubernetes.

-> Azure Service Bus fully solves every problem identified in the RabbitMQ configuration.

## Notes About Setup Challenges or Lessons Learned

Learned how Kubernetes treats stateful vs stateless applications differently.
Realized that Deployments are not suitable for databases or message brokers like RabbitMQ.
Understood the importance of PersistentVolumeClaims for durability.
Found that combining multiple resources in one YAML file can hide important configuration issues.
Experienced how LoadBalancer exposes a service externally on AKS.
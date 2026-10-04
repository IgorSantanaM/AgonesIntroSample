# Agones Intro Sample

A small hands-on project for learning the core concepts of [Agones](https://agones.dev/), an open-source platform for hosting, running, and scaling dedicated game servers on Kubernetes.

This repository uses the **Xonotic** example game server to demonstrate how to:

- deploy a standalone `GameServer`;
- manage multiple game servers with an Agones `Fleet`;
- dynamically expose game server ports;
- allocate a `Ready` server from a Fleet using `GameServerAllocation`;
- observe the lifecycle of a game server from `Ready` to `Allocated`.

> This repository is intended as a learning/demo project, not as a production-ready Agones deployment.

---

## How Agones Fits into Kubernetes

Agones extends Kubernetes with custom resources designed specifically for dedicated game servers.

```text
Player / Matchmaker
        |
        | requests a server
        v
GameServerAllocation
        |
        | selects a Ready server
        v
      Fleet
        |
        | manages
        v
+---------------------------+
| GameServer 1 - Allocated  |
| GameServer 2 - Ready      |
+---------------------------+
        |
        v
 Kubernetes Pods / Nodes
```

The basic lifecycle demonstrated in this project is:

```text
Creating -> Starting -> Scheduled -> RequestReady -> Ready -> Allocated
```

A `Ready` server is available for a game session. Once an allocation succeeds, that server changes to `Allocated`, meaning it has been reserved for a session and should no longer be offered to another player or match.

---

## Repository Structure

```text
AgonesIntroSample/
├── agonesFleet.yaml
├── fleetallocation.yaml
├── gameserver.yaml
└── README.md
```

| File | Purpose |
| --- | --- |
| `gameserver.yaml` | Creates a single standalone Xonotic `GameServer`. |
| `agonesFleet.yaml` | Creates a Fleet containing two Xonotic game servers. |
| `fleetallocation.yaml` | Allocates one `Ready` GameServer belonging to the `xonotic` Fleet. |
| `README.md` | Project documentation and walkthrough. |

---

## Prerequisites

Before running the sample, you need:

- [Docker](https://docs.docker.com/get-docker/) or another container runtime supported by your Kubernetes environment;
- [kubectl](https://kubernetes.io/docs/tasks/tools/);
- a Kubernetes cluster;
- [Agones](https://agones.dev/site/docs/installation/) installed in the cluster.

For local experimentation, [Minikube](https://minikube.sigs.k8s.io/docs/start/) is a convenient option.

Agones dynamically assigns ports to GameServers. By default, Agones uses UDP ports in the `7000-8000` range, so your cluster/network must allow the required traffic if you want to connect to the game server externally.

---

## Local Setup with Minikube

Start a Minikube cluster:

```bash
minikube start -p agones
```

Make sure `kubectl` is using the expected cluster:

```bash
kubectl config current-context
```

You can also explicitly switch to the Minikube profile:

```bash
kubectl config use-context agones
```

### Install Agones with Helm

Add the Agones Helm repository:

```bash
helm repo add agones https://agones.dev/chart/stable
helm repo update
```

Install Agones:

```bash
helm install agones \
  --namespace agones-system \
  --create-namespace \
  agones/agones
```

Verify that the Agones components are running:

```bash
kubectl get pods -n agones-system
```

> For exact Kubernetes/Minikube version compatibility, check the current [Agones Minikube installation guide](https://agones.dev/site/docs/installation/creating-cluster/minikube/).

---

## Clone the Repository

```bash
git clone https://github.com/IgorSantanaM/AgonesIntroSample.git
cd AgonesIntroSample
```

---

# 1. Create a Standalone GameServer

The simplest Agones resource is a `GameServer`.

Apply the manifest:

```bash
kubectl apply -f gameserver.yaml
```

Check the server:

```bash
kubectl get gameservers
```

Or use the shorter alias:

```bash
kubectl get gs
```

You should eventually see a result similar to:

```text
NAME      STATE   ADDRESS          PORT   NODE       AGE
xonotic   Ready   192.168.49.2     7xxx   minikube   20s
```

The important fields are:

- **STATE** — current Agones lifecycle state;
- **ADDRESS** — address of the Kubernetes node hosting the server;
- **PORT** — dynamically assigned port through which the game server is exposed;
- **NODE** — Kubernetes node running the GameServer Pod.

### What `portPolicy: Dynamic` means

The manifest defines:

```yaml
ports:
  - name: default
    portPolicy: Dynamic
    containerPort: 26000
```

The Xonotic process listens on container port `26000`, while Agones chooses an available host port dynamically and exposes it through the GameServer status.

---

# 2. Create a Fleet

Managing individual GameServers manually does not scale well. Agones provides `Fleet` to manage a group of GameServers based on the same template.

Apply the Fleet:

```bash
kubectl apply -f agonesFleet.yaml
```

The Fleet in this repository is configured with:

```yaml
spec:
  replicas: 2
```

That means Agones manages two Xonotic GameServer instances for the Fleet.

Check the Fleet:

```bash
kubectl get fleet
```

Check the GameServers created by it:

```bash
kubectl get gs
```

Example:

```text
NAME                  STATE   ADDRESS          PORT   NODE       AGE
xonotic-xxxxx-aaaaa   Ready   192.168.49.2     7536   minikube   30s
xonotic-xxxxx-bbbbb   Ready   192.168.49.2     7249   minikube   30s
```

The generated names will be different on each deployment.

You can inspect the Fleet in more detail with:

```bash
kubectl describe fleet xonotic
```

---

# 3. Allocate a GameServer

In a real multiplayer architecture, a matchmaking or session-management service normally asks Agones for an available server.

This sample demonstrates that operation with `fleetallocation.yaml`.

The allocation selects a GameServer with the Fleet label:

```yaml
selectors:
  - matchLabels:
      agones.dev/fleet: xonotic
    gameServerState: Ready
```

Create the allocation:

```bash
kubectl create -f fleetallocation.yaml -o yaml
```

Unlike the Fleet manifest, an allocation represents a request for a server. Using `kubectl create` makes it easy to submit a fresh allocation request whenever you need another server.

After the request, check the GameServers again:

```bash
kubectl get gs
```

You should see something conceptually similar to:

```text
NAME                  STATE       ADDRESS          PORT   NODE       AGE
xonotic-xxxxx-aaaaa   Allocated   192.168.49.2     7007   minikube   2m
xonotic-xxxxx-bbbbb   Ready       192.168.49.2     7380   minikube   2m
```

The states now have an important meaning:

- **Allocated** — reserved for a game session;
- **Ready** — still available for another allocation.

The allocation operation is atomic, preventing the same `Ready` GameServer from being assigned to multiple allocation requests.

---

## Inspect an Allocated Server

List the servers:

```bash
kubectl get gs
```

Inspect one in detail:

```bash
kubectl describe gs <gameserver-name>
```

You can also retrieve only its state:

```bash
kubectl get gs <gameserver-name> \
  -o jsonpath='{.status.state}'
```

Retrieve its address and port:

```bash
kubectl get gs <gameserver-name> \
  -o jsonpath='{.status.address}:{.status.ports[0].port}'
```

---

## Connecting with Xonotic

Once a GameServer is reachable from your machine, get its address and port:

```bash
kubectl get gs
```

Then, from the Xonotic client console, connect to the allocated server using the address and port reported by Agones:

```text
connect <ADDRESS>:<PORT>
```

Example:

```text
connect 192.168.49.2:7007
```

> Local networking depends on your operating system and Minikube driver. On some Windows/Minikube configurations, the node address shown by Agones is not directly reachable from the host. See the [Agones Minikube connection workarounds](https://agones.dev/site/docs/installation/creating-cluster/minikube/) if the server is running but the client cannot connect.

---

## Scaling the Fleet

You can scale the Fleet directly with Kubernetes:

```bash
kubectl scale fleet xonotic --replicas=4
```

Check the result:

```bash
kubectl get fleet
kubectl get gs
```

Scale it back down:

```bash
kubectl scale fleet xonotic --replicas=2
```

This demonstrates one of the main advantages of Agones: game server capacity can be managed declaratively through Kubernetes instead of creating dedicated server processes manually.

---

## Useful Commands

```bash
# List GameServers
kubectl get gs

# Watch GameServer state changes in real time
kubectl get gs -w

# List Fleets
kubectl get fleet

# Inspect the Xonotic Fleet
kubectl describe fleet xonotic

# Inspect a specific GameServer
kubectl describe gs <gameserver-name>

# List Pods created for the GameServers
kubectl get pods

# Scale the Fleet
kubectl scale fleet xonotic --replicas=4

# Request another allocation
kubectl create -f fleetallocation.yaml -o yaml
```

Watching GameServers is particularly useful while experimenting with allocations:

```bash
kubectl get gs -w
```

You can then submit an allocation from another terminal and immediately observe the selected server changing from `Ready` to `Allocated`.

---

## GameServer vs Fleet vs GameServerAllocation

| Resource | Responsibility |
| --- | --- |
| `GameServer` | Represents a single dedicated game server process running in Kubernetes. |
| `Fleet` | Manages a group of GameServers created from the same template. |
| `GameServerAllocation` | Finds and atomically reserves an eligible GameServer for a game session. |

A simplified production flow normally looks like this:

```text
Player queues for match
        |
        v
Matchmaker / Backend
        |
        v
GameServerAllocation
        |
        v
Agones selects Ready GameServer
        |
        v
GameServer becomes Allocated
        |
        v
Backend returns ADDRESS + PORT
        |
        v
Player connects to dedicated server
```

---

## Manifest Notes

### `gameserver.yaml`

The standalone GameServer currently uses:

```text
gcr.io/agones-images/xonotic-example:0.3
```

### `agonesFleet.yaml`

The Fleet currently uses the newer example image:

```text
us-docker.pkg.dev/agones-images/examples/xonotic-example:2.8
```

If you want identical behavior between the standalone example and the Fleet example, consider updating the standalone manifest to use the same Xonotic image version as the Fleet.

---

## Cleanup

Delete the standalone GameServer:

```bash
kubectl delete -f gameserver.yaml
```

Delete the Fleet and the GameServers it manages:

```bash
kubectl delete -f agonesFleet.yaml
```

Check that the resources are gone:

```bash
kubectl get gs
kubectl get fleet
```

If you created Minikube only for this project, you can stop it with:

```bash
minikube stop -p agones
```

Or remove the cluster completely:

```bash
minikube delete -p agones
```

---

## What This Sample Demonstrates

After completing the walkthrough, you should understand the basic relationship between:

```text
Kubernetes
   +
Agones
   |
   +-- GameServer
   |
   +-- Fleet
   |
   +-- GameServerAllocation
```

These resources form the foundation for more advanced Agones architectures involving matchmaking, autoscaling, server lifecycle management, multiple regions, observability, and production multiplayer backends.

---

## Next Steps

Possible improvements for this project include:

- add a `FleetAutoscaler`;
- connect a matchmaking/backend API to `GameServerAllocation`;
- expose allocation through a small .NET API;
- test automatic scaling based on available game-server capacity;
- add Prometheus/Grafana observability;
- deploy the sample to a multi-node Kubernetes cluster;
- automate deployment with CI/CD;
- add health, lifecycle, and shutdown demonstrations.

---

## References

- [Agones Documentation](https://agones.dev/site/docs/)
- [Create a GameServer](https://agones.dev/site/docs/getting-started/create-gameserver/)
- [Create a GameServer Fleet](https://agones.dev/site/docs/getting-started/create-fleet/)
- [GameServerAllocation Specification](https://agones.dev/site/docs/reference/gameserverallocation/)
- [Agones on Minikube](https://agones.dev/site/docs/installation/creating-cluster/minikube/)

---

## Author

**Igor Santana Medeiros**

GitHub: [@IgorSantanaM](https://github.com/IgorSantanaM)

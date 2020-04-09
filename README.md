# Istio testing in Openshift

This is a supplimentary project for following along with the Istio Doc sample projects on Openshift.
The Istio documentation uses the [httpbin](https://hub.docker.com/r/kennethreitz/httpbin/dockerfile) image which won't work without being able to run the container as root. Since Openshift automatically runs containers as random users for security reasons we need to either create our own image or just add an `anyuid` Security Context Constraint (which I have done). I also decided to automate the deployment to any number of namespaces.

## Create `SCC` and deploy the **httpbin** and **sleep** apps to namespaces
1. Login
```
oc login <mycluster>
```
2. Run playbook on cluster
```
ansible-playbook playbook.yml
```

## Test Mutual TLS (mTLS) on Service Mesh
> Expands on the example from Istio Docs [Mutual TLS Migration](https://istio.io/docs/tasks/security/authentication/mtls-migration/)

- Verify your *ServiceMeshControlPlane* has the `spec.global.mtls.enabled=true` flag set for enabling mTLS for the entire mesh. See [bookinfo](https://github.com/trevorbox/bookinfo/blob/example-1.0/service-mesh/control-plane/default-gateway/ServiceMeshControlPlane.yml) as an example.

- Verify your *ServiceMeshMemberRoll* has `foo` and `bar` listed in `spec.members` only

- Run the following to test that the `sleep.legacy` app cannot connect to `httpbin.foo` or `httpbin.bar`. Connectivity does work outside the mesh between `sleep.legacy` and `httpbin.legacy2`. Notice that traffic from `sleep.foo` or `sleep.bar` to `httpbin.legacy2` is allowed, however.
   ```
   for from in "foo" "bar" "legacy"; do for to in "foo" "bar" "legacy2"; do kubectl exec $(kubectl get pod -l app=sleep -n ${from} -o jsonpath={.items..metadata.name}) -c sleep -n ${from} -- curl http://httpbin.${to}:8000/ip -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done
   ```
   You should see the following:
   ```
   sleep.foo to httpbin.foo: 200
   sleep.foo to httpbin.bar: 200
   sleep.foo to httpbin.legacy2: 200
   sleep.bar to httpbin.foo: 200
   sleep.bar to httpbin.bar: 200
   sleep.bar to httpbin.legacy2: 200
   sleep.legacy to httpbin.foo: 000
   command terminated with exit code 7
   sleep.legacy to httpbin.bar: 000
   command terminated with exit code 7
   sleep.legacy to httpbin.legacy2: 200
   ```

# Verify Requests to test mTLS
- Follow the steps from [Mutual TLS Deep-Dive](https://archive.istio.io/v1.0/docs/tasks/security/mutual-tls/#verify-requests) to verify requests

## Helpful docs
- [Maistra mesh-wide mTLS](https://maistra.io/docs/examples/mesh-wide_mtls/)
- [Mutual TLS Deep-Dive](https://archive.istio.io/v1.0/docs/tasks/security/mutual-tls/)

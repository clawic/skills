# Commands — The kubectl Incident Toolkit

Commands that end arguments, not the basics. Every one of these answers a question you would otherwise guess at.

## Evidence Capture (do this before you change anything)

```bash
kubectl get events -A --sort-by=.lastTimestamp | tail -40   # 1h TTL — capture first
kubectl describe pod <p> > /tmp/pod.txt                     # events + probe + OOM history in one file
kubectl get pod <p> -o yaml > /tmp/pod.yaml                 # status.containerStatuses.lastState is the crash record
kubectl logs <p> -p --tail=200                              # previous container after a restart
kubectl logs <p> --all-containers --timestamps --since=15m  # sidecars included; timestamps for correlation
```

- `kubectl get event --field-selector involvedObject.name=<pod>` narrows a noisy namespace to one object.
- Events are objects, not logs: they are deduplicated with a count, and a "single" event with `count: 47` is the real story.

## Fast Situational Awareness

```bash
kubectl get pods -A --field-selector=status.phase!=Running          # everything unhappy, cluster-wide
kubectl get pods -A --sort-by=.status.containerStatuses[0].restartCount | tail
kubectl get pods -o wide                                            # node placement and pod IPs, the two columns people forget
kubectl top pods --sort-by=memory        # needs metrics-server; absence is itself a finding (autoscaling.md)
kubectl get nodes -o wide && kubectl describe node <n> | sed -n '/Allocated resources/,$p'
```

`Allocated resources` on a node prints requests vs allocatable — the exact arithmetic the scheduler used to reject a Pending pod (`scheduling.md`).

## Interrogating the API Instead of the Docs

```bash
kubectl explain deployment.spec.strategy --recursive   # matches THIS cluster's API version
kubectl api-resources --namespaced=false               # what is cluster-scoped here, CRDs included
kubectl get --raw /metrics | head                      # the API server's own view
kubectl auth can-i --list --as=system:serviceaccount:<ns>:<sa>   # effective permissions, no YAML reasoning
```

## Debugging Inside the Pod

```bash
kubectl debug -it <pod> --image=nicolaka/netshoot --target=<container>   # ephemeral container, shared PID ns (kubectl >=1.25)
kubectl debug node/<node> -it --image=busybox                            # node shell without SSH; host fs at /host
kubectl debug <pod> --copy-to=<pod>-dbg --set-image=app=busybox -- sleep 1d   # clone with a debug image, original untouched
kubectl exec <p> -c <c> -- sh -c 'cat /proc/1/environ | tr "\0" "\n"'    # what the process actually got
kubectl cp <ns>/<pod>:/path/file ./file                                  # pull a heap dump or core file out
```

- `--target` requires a runtime with process-namespace sharing; without it the ephemeral container sees its own PID 1 only.
- Distroless images have no shell — that is what ephemeral containers exist for; do not rebuild the image "with a shell for debugging" and ship it.

## Traffic and Reachability

```bash
kubectl get endpointslices -l kubernetes.io/service-name=<svc> -o wide   # the truth about wiring
kubectl port-forward svc/<svc> 8080:80        # bypasses Ingress and LB — isolates which layer is broken
kubectl run tmp --rm -it --image=nicolaka/netshoot --restart=Never -- bash
kubectl exec <p> -- getent hosts <svc>        # portable resolution test (musl and glibc both have it)
```

`port-forward` to the Service still goes to a pod; if it works and the Ingress does not, the fault is in the Ingress layer, not the app (`ingress.md`).

## Changing Things Safely

```bash
kubectl diff -f manifest.yaml                       # what would change, including defaulting and webhooks
kubectl apply -f manifest.yaml --dry-run=server     # real admission; --dry-run=client validates nothing real
kubectl rollout restart deploy/<d>                  # pick up ConfigMap/Secret changes delivered as env vars
kubectl rollout undo deploy/<d> --to-revision=3     # needs the old ReplicaSet (revisionHistoryLimit, default 10)
kubectl scale deploy/<d> --replicas=0 --timeout=60s # the honest stop; deleting pods just recreates them
kubectl annotate pod <p> debug=keep --overwrite     # mark what you are inspecting so a teammate does not delete it
```

## The Two Escape Hatches (and their price)

```bash
kubectl delete pod <p> --grace-period=0 --force     # API forgets it; kubelet may still be running it
kubectl patch pod <p> -p '{"metadata":{"finalizers":null}}' --type=merge
```

Both leave the cluster lying to you about reality. Preconditions before either: the owning controller is understood, and for force-delete of a StatefulSet member the node is confirmed dead (`operators.md`, `stateful.md`).

## Output Plumbing Worth Memorizing

```bash
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
kubectl get pod <p> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'   # OOMKilled or Error
kubectl get deploy -A -o json | jq -r '.items[] | select(.spec.template.spec.containers[].resources.limits.memory == null) | "\(.metadata.namespace)/\(.metadata.name)"'
```

The last one is the fleet audit that finds every workload one burst away from evicting a neighbor (`resources.md`).

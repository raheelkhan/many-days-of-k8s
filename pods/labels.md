# Labels
labels are use to identify objects.

To create pod with label 
```bash
kubectl run hazelcast --image=hazelcast/hazelcast --labels="app=hazelcast,env=prod"
```

Lets create another pod with a different label
```bash
kubectl run hazelcast --image=hazelcast/hazelcast --labels="app=hazelcast,env=dev"
```

Lets create another pod with a different label
```bash
kubectl run hazelcast-canary --image=hazelcast/hazelcast --labels="app=hazelcast,env=dev,canary=true"
```

In order to get all pods with `env=prod` label I can do
```bash
k get pods --selector="env=prod"
```

To get all pods where both `app=hazlecast` and `env=prod`
```bash
k get pods --selector="app=hazelcast,env=prod"
```

We can also get OR condition
```bash
k get pods --selector="env in (prod, dev)"
```

Lets say if I want to get only pod that has a label key called `canary` regardless of value
```bash
k get pods --selector="canary"
```

## Operators
- key=value key is set to value
- key!=value key is not set to value
- key in (value1, value2) value of key must in either value1 or value2
- key notin (value1, value2) value of key must not be in either value1 or value2
- key if we only supply key, it means that select anything with this key present and do not check the value
- !key means select anything where this key is not present

```bash
k get pods --selector='!canary' # it only works with single quotes
```

For the practice of using selectors in a yaml file I have created a file `practice-labels-selectors.yaml`

In the above file I tried following scenarios

> match all pods with label `app.kubernetes.io/name` equals to `myapp`

> match all pods with label `app.kubernetes.io/name` equals to `myapp` but the key `app.kubernetes.io/env` must not equals to `dev`

I was not able to do this set based selection because it is not supported in `Service` object.
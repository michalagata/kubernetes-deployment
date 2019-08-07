Użycie publicznego obrazu dockera (nginx) do testu.

* Uruchomienie deploymentu

`kubectl run testnginx --image=nginx --replicas=2 --port=80`

`kubectl get deployments`

```
NAME       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
testnginx   2         2         2            2           1m
```

`kubectl get pods`

```
NAME                        READY     STATUS    RESTARTS   AGE
testnginx-xxxxxxxxxx-vsv1   1/1       Running   0          1m
testnginx-xxxxxxxxxx-in1x   1/1       Running   0          1m
```

* jeśli jest gotowy, wystawiamy na świat

`kubectl expose deployment testnginx --type="LoadBalancer"`

`root@kube-master:/usr/local/bin# kubectl get services`

```
NAME         CLUSTER-IP        EXTERNAL-IP   PORT(S)   AGE
kubernetes   192.168.0.1       <none>        443/TCP   53m
testnginx     192.168.176.186   <pending>     80/TCP    12s
```

* Testujemy dostęp przez flannel

`curl -I 192.168.176.186`
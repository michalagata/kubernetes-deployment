`kubectl create -f https://rawgit.com/kubernetes/dashboard/master/src/deploy/kubernetes-dashboard.yaml`

* Sprawdzanie postępu oraz zakończenia instalacji poprzez:

`kubectl get --namespace="kube-system" pods`
`kubectl get --namespace="kube-system" deployments`
`kubectl get --namespace="kube-system" services`

również `docker ps` na poziomie odnóg powinien zwrócić informacje o odpaleniu

po zakończeniu, wejść na adres dashboardu http://10.1.254.80:8080/ui/ 

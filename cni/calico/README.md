## Calico

- Установите CNI (Container Network Interface) Calico на master-ноде:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.5/manifests/calico.yaml
```

- Проверьте что все ноды кластера имеют статус Ready с помощью команды:

```bash
kubectl get no -o wide
```

- Проверьте что все системные поды кластера имеют статус Ready и счетчик Restart не увиличивается с помощью команды:

```bash
kubectl get po -n kube-system
```
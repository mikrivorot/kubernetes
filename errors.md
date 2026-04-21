## 1.
```
NAME                      READY   STATUS             RESTARTS        AGE
grade-submission-api      1/2     ImagePullBackOff   0               7s
grade-submission-portal   2/2     Running            2 (4d12h ago)   10d
```

If READY 1/2 - this pod have two containers, one is running and the other is not (in ImagePullBackOff state)


## 2.
```
ConnectionError
requests.exceptions.ConnectionError: HTTPConnectionPool(host='localhost', port=3000): Max retries exceeded with url: /grades (Caused by NewConnectionError('<urllib3.connection.HTTPConnection object at 0x7f9a03c7e7f0>: Failed to establish a new connection: [Errno 111] Connection refused'))
```

Get code of your app to check variables:
```
kubectl exec grade-submission-portal -- cat /app/app.py
```
after mmodifications:
```
kubectl delete pod grade-submission-portal
kubectl apply -f service_discovery/grade-submission-portal-pod.yaml
kubectl get pods,services
```
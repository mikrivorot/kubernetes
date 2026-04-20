```
NAME                      READY   STATUS             RESTARTS        AGE
grade-submission-api      1/2     ImagePullBackOff   0               7s
grade-submission-portal   2/2     Running            2 (4d12h ago)   10d
```

If READY 1/2 - this pod have two containers, one is running and the other is not (in ImagePullBackOff state)
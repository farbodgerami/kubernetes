
### Create Mariadb cluster:
```
helm install mariadb bitnami/mariadb --namespace mariadb  -f values.yaml --create-namespace
```

### Create Mongodb cluster:
```
helm install mongodb bitnami/mongodb --namespace mongodb  -f values.yaml --create-namespace
```

### Create Redis cluster:
```
helm install redis bitnami/redis --namespace redis  -f values.yaml --create-namespace
```

### Create Rabbitmq cluster:
```
helm install rabbitmq bitnami/rabbitmq --namespace rabbitmq  -f values.yaml --create-namespace
```
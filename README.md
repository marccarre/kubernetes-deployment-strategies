# Kubernetes deployment strategies & databases

## Instructions

### Create a Kubernetes cluster

- Install minikube & start your local cluster.

### Deploy the DB and web service

- Run `kubectl apply -f db`
- Run `kubectl apply -f web`
- You should see the below pods being created:

        $ kubectl get po
        NAME                              READY     STATUS              RESTARTS   AGE
        kds-postgresql-844c696487-4trsg   0/1       Running             0          4s
        kds-service-7dd564bdc-k9kpl       0/1       ContainerCreating   0          2s
        kds-service-7dd564bdc-s8fsx       0/1       ContainerCreating   0          2s
        kds-service-7dd564bdc-xqnbn       0/1       ContainerCreating   0          2s

- After a few seconds, you should then see:

        $ kubectl get po
        NAME                              READY     STATUS    RESTARTS   AGE
        kds-postgresql-844c696487-4trsg   1/1       Running   0          25s
        kds-service-7dd564bdc-k9kpl       1/1       Running   0          23s
        kds-service-7dd564bdc-s8fsx       1/1       Running   0          23s
        kds-service-7dd564bdc-xqnbn       1/1       Running   0          23s

- Run `minikube addons enable ingress`
- Run `minikube service kds-service`

### Talk to the service

```console
$ curl -fsS "$(minikube service kds-service --url)/users" | jq
[]

$ curl -i -X POST \
  -H "Content-Type: application/json" \
  --data '{"firstName":"Luke","familyName":"Skywalker"}' \
  "$(minikube service kds-service --url)/users"

HTTP/1.1 201 Created
Location: /users/1
Date: Mon, 16 Jul 2018 10:46:42 GMT
Content-Length: 0

$ curl -fsS "$(minikube service kds-service --url)/users" | jq
[
  {
    "id": 1,
    "firstName": "Luke",
    "familyName": "Skywalker"
  }
]

$ curl -fsS "$(minikube service kds-service --url)/users/1" | jq
{
  "id": 1,
  "firstName": "Luke",
  "familyName": "Skywalker"
}
```

## Misc.

### Generate the DB's manifests

The manifests under `./db` were generated using Helm.
In case you need or want to reproduce these steps, these are:

- Install Helm.
- Run: `helm init`
- Run: `helm install --name kds stable/postgresql`
- Run: `helm get manifest kds > ./db/postgresql.yaml`
- Change `POSTGRES_DB` to be set to `users`

### Querying the DB

```
$ DB_POD_ID="$(kubectl get po | grep kds-postgresql | awk '{print $1}')"

$ kubectl exec -it "$DB_POD_ID" -- psql -U postgres users
psql (9.6.2)
Type "help" for help.

users=# \dt
               List of relations
 Schema |       Name        | Type  |  Owner
--------+-------------------+-------+----------
 public | schema_migrations | table | postgres
 public | users             | table | postgres
(2 rows)

users=# SELECT * FROM users;
 id | first_name | family_name
----+------------+-------------
(0 rows)
```

# Kubernetes deployment strategies & databases

## Instructions

### Create a Kubernetes cluster

- Install minikube & start your local cluster.

### Install Weave Cloud

- Go to [cloud.weave.works](https://cloud.weave.works).
- Click on "_+Connect a cluster_".
- Select "_Kubernetes_" and then "_Minikube_".
- Run the command provided.
- Wait for all agents to start & connect to Weave Cloud. This may take a few minutes.
- Click "_View your cluster_"

### Configure Weave Cloud for Continuous Delivery

- Click on "_Deploy_" and then "_Configure_".
- Fork this repository.
- Configure "_Deploy_" to point to your repository & follow the instructions.

Once done, Weave Cloud will synchronise your Kubernetes cluster with your repository.
You should eventually see in your Kubernetes cluster a PostgreSQL DB and a web service using it:
```console
$ kubectl get po
NAME                              READY     STATUS    RESTARTS   AGE
kds-postgresql-7cc4658b5f-vqkvt   1/1       Running   0          20s
kds-service-f8ddcdc68-99hdd       1/1       Running   0          20s
kds-service-f8ddcdc68-hwcbj       1/1       Running   0          20s
kds-service-f8ddcdc68-krd8h       1/1       Running   0          20s
```

That's all folks! Welcome to GitOps with Weave Cloud!

### Deploy the DB and web service without Weave Cloud

If you do NOT want to use Weave Cloud, you can still run this demo by deploying manually instead.
Skip the "_Install Weave Cloud_" and "_Configure Weave Cloud_" steps above, and do the following:

- Run `kubectl apply -f db`
- Run `kubectl apply -f web`
- You should see the below pods being created:

  ```console
  $ kubectl get po
  NAME                              READY     STATUS              RESTARTS   AGE
  kds-postgresql-7cc4658b5f-vqkvt   0/1       Running             0          4s
  kds-service-f8ddcdc68-99hdd       0/1       ContainerCreating   0          2s
  kds-service-f8ddcdc68-hwcbj       0/1       ContainerCreating   0          2s
  kds-service-f8ddcdc68-krd8h       0/1       ContainerCreating   0          2s
  ```

- After a few seconds, you should then see:

  ```console
  $ kubectl get po
  NAME                              READY     STATUS    RESTARTS   AGE
  kds-postgresql-7cc4658b5f-vqkvt   1/1       Running   0          25s
  kds-service-f8ddcdc68-99hdd       1/1       Running   0          23s
  kds-service-f8ddcdc68-hwcbj       1/1       Running   0          23s
  kds-service-f8ddcdc68-krd8h       1/1       Running   0          23s
  ```

### Test our deployment

```console
$ curl -fsS "$(minikube service kds-service --url)" | jq
[
  {
    "method": "GET",
    "path": "/"
  },
  {
    "method": "GET",
    "path": "/healthz"
  },
  {
    "method": "POST",
    "path": "/users"
  },
  {
    "method": "GET",
    "path": "/users"
  },
  {
    "method": "GET",
    "path": "/users/{id:[0-9]+}"
  }
]

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

```console
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

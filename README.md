# Kubernetes deployment strategies & databases

## Instructions

### Create a Kubernetes cluster

- Install [minikube](https://kubernetes.io/docs/setup/minikube/#installation) & start your local cluster.

### Install Weave Cloud

- Go to [cloud.weave.works](https://cloud.weave.works).
- Click on "_+Connect a cluster_".
- Select "_Kubernetes_" and then "_Minikube_".
- Run the command provided.
- Wait for all agents to start & connect to Weave Cloud. This may take a few minutes.
- Click "_View your cluster_"

### Enable GitOps

- Click on "_Deploy_" and then "_Configure_".
- Fork this repository.
- Configure "_Deploy_" to point to your repository & follow the instructions.

Once done, Weave Cloud will synchronise your Kubernetes cluster with your repository.
You should eventually see in your Kubernetes cluster a PostgreSQL DB and a web service using it:

```console
$ kubectl get po
NAME                              READY     STATUS    RESTARTS   AGE
kds-postgresql-84798fd5f-gmbl4     1/1       Running   0          20s
kds-service-847fb558b8-00001       1/1       Running   0          20s
kds-service-847fb558b8-00002       1/1       Running   0          20s
kds-service-847fb558b8-00003       1/1       Running   0          20s
```

That's all folks! Welcome to GitOps with Weave Cloud!

### Test our deployment

At this stage, we have deployed `kds-service:v1.0.0` which can store users' first names and family names.
We package database migrations within the `kds-service`pod, therefore our database's schema has been automatically created during the rollout of the service:

```console
$ kubectl logs kds-service-847fb558b8-00001
level=info msg="upgrading DB schema..." currentVersion=0 targetVersion=1
$ kubectl logs kds-service-847fb558b8-00002
level=info msg="nothing to do: DB already at or above the target schema version" currentVersion=1 targetVersion=1
$ kubectl logs kds-service-847fb558b8-00003
level=info msg="nothing to do: DB already at or above the target schema version" currentVersion=1 targetVersion=1
```

Let's try it out!

```console
$ curl -fsS "$(minikube service kds-service --url)/users" | jq
[]

$ curl -i -X POST \
  -H "Content-Type: application/json" \
  --data '{"firstName":"Luke","familyName":"Skywalker"}' \
  "$(minikube service kds-service --url)/users"

HTTP/1.1 201 Created
Location: /users/1
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

Great, we have successfully:

- created a new user & saved it in our database,
- read all users back (just one for now),
- read only the user we created.

### Enable Continuous Delivery

Next, we will enable Continuous Delivery in Weave Cloud:

- Go to [cloud.weave.works](https://cloud.weave.works).
- Click on "_Deploy_", select `kds-service` and then click "_Automate_".

After a few seconds, Weave Cloud will detect the latest version of `kds-service`, `v1.1.0` and will deploy it automatically.
As we are doing GitOps, note that this change is reflected in your Git repository.

### Test our deployment

At this stage, we have deployed `kds-service:v1.1.0` which can store users' first names, family names, **and ages**.
Again, since we package database migrations within the `kds-service`pod, our database's schema has been automatically upgraded during the rollout of the newer version of our service, with an additional `age` column in our `users` table.

```console
$ kubectl logs kds-service-57677f44d7-00001
level=info msg="upgrading DB schema..." currentVersion=1 targetVersion=2
$ kubectl logs kds-service-57677f44d7-00002
level=info msg="nothing to do: DB already at or above the target schema version" currentVersion=2 targetVersion=2
$ kubectl logs kds-service-57677f44d7-00003
level=info msg="nothing to do: DB already at or above the target schema version" currentVersion=2 targetVersion=2
```

Let's try it out!

```console
$ curl -fsS "$(minikube service kds-service --url)/users" | jq
[
  {
    "id": 1,
    "firstName": "Luke",
    "familyName": "Skywalker",
    "age": 0
  }
]

$ curl -i -X POST \
  -H "Content-Type: application/json" \
  --data '{"firstName":"Obi-Wan","familyName":"Kenobi","age":40}' \
  "$(minikube service kds-service --url)/users"

HTTP/1.1 201 Created
Location: /users/2
Content-Length: 0

$ curl -fsS "$(minikube service kds-service --url)/users" | jq
[
  {
    "id": 1,
    "firstName": "Luke",
    "familyName": "Skywalker",
    "age": 0
  },
  {
    "id": 2,
    "firstName": "Obi-Wan",
    "familyName": "Kenobi",
    "age": 40
  },
]

$ curl -fsS "$(minikube service kds-service --url)/users/2" | jq
{
  "id": 2,
  "firstName": "Obi-Wan",
  "familyName": "Kenobi",
  "age": 40
}
```

Great, like before, we have successfully:

- created a new user & saved it in our database,
- read all users back (the one we had previously saved, which has the default age `0`, and the one we just created),
- read only the user we created.

### Rolling back

Maybe storing the `age` is actually something we have second thoughts about.
Or maybe we want to roll this change back, for whatever reason.

- Go to [cloud.weave.works](https://cloud.weave.works).
- Click on "_Deploy_", select `kds-service` and then click "_De-Automate_".
- Click on "_Release_" next to "_v1.0.0_", our previous version.

That's all folks! We have just rolled back to our previous version.

What about the database?
- No migration is run in our case:
  ```console
  $ kubectl logs kds-service-847fb558b8-00004
  level=info msg="upgrading DB schema..." currentVersion=2 targetVersion=1
  $ kubectl logs kds-service-847fb558b8-00005
  level=info msg="upgrading DB schema..." currentVersion=2 targetVersion=1
  $ kubectl logs kds-service-847fb558b8-00006
  level=info msg="upgrading DB schema..." currentVersion=2 targetVersion=1
  ```
- Since our schema & code changes are backward compatible, we can still query & create users:
  ```console
  $ curl -fsS 35.225.108.173:8080/users | jq
  [
    {
      "id": 1,
      "firstName": "Luke",
      "familyName": "Skywalker"
    },
    {
      "id": 2,
      "firstName": "Obi-Wan",
      "familyName": "Kenobi"
    }
  ]
  ```
  N.B.: The age field is not returned, even though still present in database. Rolling forward again would make them being returned.

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

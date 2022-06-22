# Backups of Kubernetes Stateful Applications
Instructions for getting your backups up and running for persistent data from stateful apps on k8s. I'll be using PostgreSQL running on a small kind cluster, with a Persistent Volume, as the example we're backing up. This was tutorial was designed from my personal lab notes while backing up Postgres for NextCloud. Really you just need the bitnami postgres helm chart installed and you should be fine.

I was going to use velero, but then I read a bunch of posts saying it was [slower](https://www.reddit.com/r/kubernetes/comments/u1uqip/comment/i4fflnc/?utm_source=share&utm_medium=web2x&context=3), and often incompatible with alternative block storage, and since I want to backup to Backblaze b2, that leaves me in a bit of a pickle, until I found [k8up](https://github.com/k8up-io/getting-started). I'm so sorry you have to memorize another 3-4 letter k8s app... I'm also in pain.

K8up seems pretty awesome though, and it wraps Restic. [Restic](https://restic.net/) has a good reputation, and consistent mention in the k8s community as well, so much so that k8up is not the only app written around restic. There's also [Stash](https://stash.run/), but stash is expensive for production workloads, and gates you out of a number of features, so we're going to ignore them for now, because k8up is free and we love free things.

# Getting started
I'm assuming you already have a cluster up and running and at least one namespace for an additional stateful app you want to monitor, which in my case is nextcloud. If you don't have a cluster, check out my basic cluster intro [here](https://github.com/jessebot/argo-vault-example).

Install k8up on your cluster
```bash
# Create a namespace to house it all
kubectl create k8up

# need the CRDs
kubectl apply -f https://github.com/k8up-io/k8up/releases/download/v2.3.0/k8up-crd.yaml --namespace k8up

helm repo add appuio https://charts.appuio.ch
helm repo update

# `k8sup-backups` can be anything; can also omit & use `--generate-name` instead
helm install k8sup-backups appuio/k8up --namespace k8up
```

## Grab helm charts for k8s gitops
For a local repo where you keep your `values.yaml` and `Chart.yaml` together, you'll wanna grab it from k8up's helm chart [repo](https://github.com/appuio/charts/tree/master/appuio/k8up). I used to think you did it like this, which would mean you would need to go find the git repo:
```bash
# Chart.yaml
wget https://raw.githubusercontent.com/appuio/charts/master/appuio/k8up/Chart.yaml

# values.yaml
wget https://raw.githubusercontent.com/appuio/charts/master/appuio/k8up/values.yaml
```
But apparently there's another way.
```bash
TODO: put the other way
helm
```

And then if you modify anything in `helm/*` you'll need to run:
```bash
helm dep update helm/
git commit -m "updating helm chart: reason for updating" helm
```

## Backblaze b2 bucket
If you haven't already, create a bucket for these backups, which you can do via Backblaze's api/cli, but I did through the [Web UI](https://help.backblaze.com/hc/en-us/articles/1260803542610-Creating-a-B2-Bucket-using-the-Web-UI), and there's no shame in it. This isn't an interview.

Then, also if you haven't already, create an application key for restic, also using the [Web UI](https://help.backblaze.com/hc/en-us/articles/360052129034-Creating-and-Managing-Application-Keys). No Shame :triumph:

## Create K8s Secrets for Restic Access to your Bucket
K8up will run the restic commands for you, so you don't even actually need the restic cli tool locally to initialize the repo like you would noramlly need to with restic.

*Note *: the following secrets need to live in the same namespace as the application you want to backup. That's why I'm putting my secrets in the nextcloud namespace, because my postgres pod runs there.

Create a k8s secret for the Restic repo secret specifically:
```bash
# k8up-restic-b2-repo-pw is just the name I used in my `backup.yaml`/`schedule.yaml`
# can be anything as long as all match in secret and backup/schedule resources
 kubectl create secret generic k8up-restic-b2-repo-pw  --from-literal=password=$YOUR_PASSWORD_HERE --namespace nextcloud
```
*tip*: if you put a space before the command, bash won't save it in history

Create a secret with your Backblaze b2 application key id and application key for your bucket like:
```bash
# k8up-restic-b2-creds-nextcloud-pg is just the name I used, but
# can be anything as long as all match in secret and backup/schedule resources
 kubectl create secret generic k8up-restic-b2-creds-nextcloud-pg --from-literal=application-key-id=$YOUR_KEY_ID_HERE --from-literal=application-key=$YOUR_KEY_HERE --namespace nextcloud
```
There's definitely cooler vault based ways to get some of this done, but my vault docs are under construction, so we're doing it the old fashion way, less great, but not the worst.

## Schedule Restic Backups with k8up Schedule CRD

### Application Aware Backups
According to the [k8up docs](https://k8up.io/k8up/2.3/how-tos/application-aware-backups.html#_postgresql), You'll need to annotate your pods you want backed up with `k8up.io/backup` and in my case, since I'm using postgres, I'd need to add this to my annotations on the pod:
```yaml
      k8up.io/backupcommand: sh -c 'PGDATABASE="$POSTGRES_DB" PGUSER="$POSTGRES_USER" PGPASSWORD="$POSTGRES_PASSWORD" pg_dump --clean'
      k8up.io/file-extension: .sql
```

#### Annotate via kubectl
You could literally do a `k edit` on this, which throws you into your default text editor, normally vi/vim, or you could knock it out entirely via the command line. For the edit, assuming nextcloud with postgres, you want something like:
```
kubectl edit pod nextcloud-postgresql-0 --namespace nextcloud
```
Then find the metadata section, and past the above yaml annotations directly in in there before saving and exiting.

Another example for annotating a postgres nextcloud pod, assuming you have `POSTGRES_DB`, `POSTGRES_PASSWORD`, and `POSTGRES_USER` as env variables configured for the pod already.
```bash
kubectl annotate pods nextcloud-postgresql-0 k8up.io/file-extension=.sql --namespace nextcloud
kubectl annotate pods nextcloud-postgresql-0 k8up.io/backupcommand="sh -c 'PGDATABASE="$POSTGRES_DB" PGUSER="$POSTGRES_USER" PGPASSWORD="$POSTGRES_PASSWORD" pg_dump --clean'" --namespace nextcloud
```

#### Annotate via Helm
If you're using the postgres helm chart, or a helm chart that chains postgres, you can do something like this in your `values.yaml`:
```yaml
## PostgreSQL chart configuration
## for more options see https://github.com/bitnami/charts/tree/master/bitnami/postgresql
postgresql:
  enabled: true
  global:
    postgresql:
      auth:
        username: nextcloud
        password: areallycoolpassword
        database: nextcloud
  primary:
    annotations:
      k8up.io/backupcommand: sh -c 'PGDATABASE="$POSTGRES_DB" PGUSER="$POSTGRES_USER" PGPASSWORD="$POSTGRES_PASSWORD" pg_dump --clean'
      k8up.io/file-extension: .sql
    persistence:
      enabled: true
```

### Create a one time backup to b2
Make sure the bucket parameters in `backup.yaml`/`schedule.yaml` are set to your bucket. I've left `nextcloud` here as an example:
```yaml
    b2:
      bucket: nextcloud
```

```bash
# create the backup `schedule` resource
k apply -f backup-yamls/backup.yaml
```
You can find a further explanation on how to do this with minio in the [k8up docs](https://k8up.io/k8up/2.3/how-tos/backup.html).

### Create a scheduled backup to b2
Create the aforementioned `schedule.yaml` to backup once a day and prune monthly.
```bash
# create the backup `schedule` resource
k apply -f backup-yamls/schedule.yaml
```

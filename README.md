# Backups of k8s stateful apps
Instructions specific to MacOS and KIND for getting your backups up and running for persistent data, with nextcloud postgres as the example we're backing up.

<strike>Currently, bitnami is the standard location to get a postgresql helm chart, and their docs say to use VmWare Tanzu's Velero, so that's what we're using. Their tutorial is [here](https://docs.bitnami.com/tutorials/migrate-data-bitnami-velero/).</strike>

I was gonna use velero, then I read a bunch of forum posts saying it was [slower](https://www.reddit.com/r/kubernetes/comments/u1uqip/comment/i4fflnc/?utm_source=share&utm_medium=web2x&context=3) and often incompatible with alternative block storage stuff, and since I want ot backup to backblaze b2, that leaves me in a bit of a pickle, until I found [k8up](https://github.com/k8up-io/getting-started). I'm so sorry you have to memorize another 3-4 letter k8s app... I hate this too.

K8up seems to be based on restic, and restic really is a good and consistent mention in the k8s community, so much so that k8up is not the only app written around and capitalizing on restic. There's also Stash, but stash is expensive for production workloads and gates you out of a number of features, so we're going to ignore them for now.

# Getting started
Install k8up on your cluster
```bash
# Create a namespace to house it all
kubectl create k8up

# need the CRDs
kubectl apply -f https://github.com/k8up-io/k8up/releases/download/v2.3.0/k8up-crd.yaml --namespace k8up

helm repo add appuio https://charts.appuio.ch
helm repo update

# `k8sup-21jun22` can be anything; can also omit & use `--generate-name` instead
helm install k8sup-21jun22 appuio/k8up --namespace k8up
```

## helm chart for argo and the like
For a local repo where you keep your `values.yaml` and `Chart.yaml` together, you'll wanna grab it from k8up's helm chart [repo](https://github.com/appuio/charts/tree/master/appuio/k8up), like so:
```bash
# Chart.yaml
wget https://raw.githubusercontent.com/appuio/charts/master/appuio/k8up/Chart.yaml

# values.yaml
wget https://raw.githubusercontent.com/appuio/charts/master/appuio/k8up/values.yaml
```

And then if you fork this repo (and/or if you want to contribute back), and you modify anything in `helm/*` you'll need to run:
```bash
helm dep update helm/
git commit -m "updating helm chart: reason for updating" helm
```

# install and run restic
Read more for your distro [here](https://restic.readthedocs.io/en/latest/020_installation.html):
```bash
# installation
brew install restic
```

Then you'll need to initialize a restic repo stored at backblaze.
*Note: restic can automatically create the bucket if it doesn't exist, but only if the application key you provided can create buckets*
```bash
# export your variables for the bucket auth
export B2_ACCOUNT_ID=$YOUR_KEY_ID_HERE
export B2_ACCOUNT_KEY=$YOUR_KEY_HERE

# nextcloud-pgsql is just the name of my bucket, but you can use anything
restic -r b2:nextcloud-pgsql:default init
```
Then it should ask you for a password and you need to keep track of that, for your `schedule.yaml` that we'll be configuring below.

You'll want to also make sure you have a k8s secret for that restic repo password you created, so let's get that done now:
```bash
# restic-repo is just the name I used in my `schedule.yaml`, can be anything as long as both match in secret and backup resource
kubectl create secret generic restic-backup-repo --from-literal=password=$YOUR_PASSWORD_HERE --namespace k8up
```

# backup to backblaze b2
Create a secret with your application id and application key like. `b2-credentials-pgsql` is just the name I used for my k8s secret in my `schedule.yaml`, can be anything as long as both match in secret and backup resource
```bash
kubectl create secret generic b2-credentials-pgsql --from-literal=application-key-id=$YOUR_KEY_ID_HERE --from-literal=application-key=$YOUR_KEY_HERE --namespace k8up
```

Create the aforementioned `schedule.yaml`. You can find a further explanation how to do this with minio in the [k8up docs](https://k8up.io/k8up/2.3/how-tos/backup.html).
```bash
# create the backup `schedule` resource
k apply -f charts/k8up/schedule.yaml
```

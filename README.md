# kpt packages for use with Flux

What is this? A set of prepared packages for using with Flux.

## How to use the packages

In general:

```sh
$ kpt pkg get https://github.com/squaremo/models.git/<model>
# ...
$ kpt cfg list-setters ./<model>
# ...
$ kpt cfg set <model> ./<model> <setter> <value>
```

Specifically:

### How to bootstrap clusters with the flux-readonly model

Create a repo for the bootstrap config:

```sh
$ mkdir bootstrap
$ cd bootstrap
$ git init
# You will need to create an empty repo in github, and set your own URL here
$ REPO_BOOTSTRAP=https://github.com/squaremo/bootstrap
# ... then use the SSH equivalent here
$ git remote add origin ssh://git@github.com/squaremo/bootstrap
# ... finally, just to get any commits github did for you
$ git pull origin master
```

Now to add a model for Flux to self-sustain:

```sh
# This one is fixed -- you don't supply your own.
$ REPO_MODELS=https://github.com/squaremo/flux-models.git
$ kpt pkg get $REPO_MODELS/flux-readonly flux
fetching package /flux-readonly from https://github.com/squaremo/flux-models to flux
$ git add flux
$ git commit -m "Add flux-readonly model as flux/"
$ kpt cfg list-setters ./flux
   NAME                     VALUE                    SET BY   DESCRIPTION   COUNT
  git-url   git@github.com:fluxcd/flux-get-started                          1
$ kpt cfg set ./flux git-url $REPO_BOOTSTRAP
set 1 fields
$ git add -u
$ git commit -m "Set Flux git URL to this repository"
$ git push origin master
# ...
```

Now, if you apply the configuration to a cluster, you will bootstrap a
git-syncing process that applies Flux and whatever else you put in
your bootstrap repo:

```sh
$ kubectl apply -f ./flux
# ...
$ kubectl logs -n flux-system deploy/flux -f
# logs ...
# logs ...
# logs ...
```

### How to run the ArgoCD model

Once you have a self-sustaining Flux installation, you can add things
to the repo and they will be applied. For example, the ArgoCD model:

```
$ kpt pkg get $REPO_MODELS/argocd argocd
fetching package /argocd from https://github.com/squaremo/flux-models to argocd
$ git add argocd
$ git commit -m "Added argocd model as argocd/"
$ kpt cfg list-setters ./argocd
    NAME          VALUE       SET BY   DESCRIPTION   COUNT
  namespace   argocd-system                          28
$ # Nothing needs changing
$ git push origin master
```

(NB the model runs ArgoCD in the namespace `argocd-system`, which is
different to the regular ArgoCD installation. This affects the
port-forward later on.)

It might take a while for Flux to sync it, since it's just polling
(you could set up [webhooks](https://github.com/fluxcd/flux-recv) --
maybe later). You can watch the logs to see when something happens:

```sh
$ kubectl logs -n flux-system deploy/flux -f
# logs logs logs ...
ts=2020-05-12T05:33:04.7525229Z caller=loop.go:133 component=sync-loop event=refreshed url=https://@github.com/squaremo/flux-models-bootstrap branch=master HEAD=d1b2d98c951d28cff9cbf8344c44d908e91c9b09
```

Now you should be able to bring up the ArgoCD user interface, by
running a port-forward:

```sh
$ kubectl port-forward svc/argocd-server -n argocd-system 8083:443 &
```

.. and open a browser tab on https://localhost:8083 (it may complain
that the server certificate is not signed for localhost, which you'll
have to click through).

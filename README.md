<!-- note marker start -->
**NOTE**: This repo contains only the documentation for the private BoltsOps Pro repo code.
Original file: https://github.com/boltopspro/ruby-frontend/blob/master/README.md
The docs are publish so they are available for interested customers.
For access to the source code, you must be a paying BoltOps Pro subscriber.
If are interested, you can contact us at contact@boltops.com or https://www.boltops.com

<!-- note marker end -->

# Demo Ruby Frontend Project

[![BoltOps Badge](https://img.boltops.com/boltops/badges/boltops-badge.png)](https://www.boltops.com)

This project contains a small sinatra app that prints out some text.  It listens on the default 8100 port.

The frontend app makes a network call to the backend app. The backend app has a NetworkPolicy that only allows pods in the ruby-backend and ruby-frontend namespace to talk to it.

![](https://img.boltops.com/boltopspro/demo-apps/frontend/frontend-to-backend.png)

This demo project demonstrates kubes. More info here: https://kubes.guru

## Testing Locally with Mac OSX

    $ git clone https://github.com/boltopspro-docs/ruby-frontend
    $ cd ruby-frontend
    $ bundle
    $ ruby app.rb

## Testing Locally with Docker

The app is also dockerized so you can test this via docker. Note, the ruby-backend app needs to be running first.

    $ docker build -t ruby-frontend .
    $ docker run --rm -d -p 8101:8101 --name ruby-frontend --link ruby-backend -e BACKEND_ENDPOINT=http://ruby-backend:8100 ruby-frontend:latest
    $ docker ps # to confirm running container
    CONTAINER ID        IMAGE                  COMMAND             CREATED             STATUS              PORTS                    NAMES
    769165d73133        ruby-frontend:latest   "bin/web"           7 seconds ago       Up 6 seconds        0.0.0.0:8101->8101/tcp   ruby-frontend
    22b3c7442c46        ruby-backend:latest    "bin/web"           11 seconds ago      Up 10 seconds       0.0.0.0:8100->8100/tcp   ruby-backend
    $ curl localhost:8101
    frontend calling backend. response from backend: message from backend
    $ docker stop ruby-frontend
    $

You can see the message: `frontend calling backend. response from backend: message from backend`

## Deploying

Note: You'll need to have a docker repo that you can push to set up. The example here uses a google GCP repo.

    $ kubes deploy
    => docker build -t gcr.io/google_project/ruby-frontend:kubes-2020-07-16T21-42-54-9bd747c -f Dockerfile .
    Successfully built 0ddeb2d63561
    Successfully tagged gcr.io/google_project/ruby-frontend:kubes-2020-07-16T21-42-54-9bd747c
    => docker push gcr.io/google_project/ruby-frontend:kubes-2020-07-16T21-42-54-9bd747c
    Pushed gcr.io/google_project/ruby-frontend:kubes-2020-07-16T21-42-54-9bd747c docker image.
    Docker push took 4s.
    Compiled  .kubes/resources files to .kubes/output
    Deploying kubes resources
    => kubectl apply -f .kubes/output/shared/namespace.yaml
    namespace/ruby-frontend created
    => kubectl apply -f .kubes/output/web/service.yaml
    service/web created
    => kubectl apply -f .kubes/output/web/deployment.yaml
    deployment.apps/web created
    $

## Test

List the frontend resources:

    $ kubectl config set-context --current --namespace=ruby-frontend # kubens ruby-frontend
    $ kubectl get pod,svc
    NAME                       READY   STATUS    RESTARTS   AGE
    pod/web-5f9dddd964-mtzgz   1/1     Running   0          95s

    NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
    service/web   ClusterIP   192.168.2.38   <none>        80/TCP    95s

Grab the pod name and exec into it.

    $ kubectl exec -ti pod/web-5f9dddd964-mtzgz sh
    /app # curl web.ruby-backend # checking backend first
    message from backend
    /app # curl web.ruby-frontend # checking frontend -> backend
    frontend calling backend. response from backend: message from backend
    /app # curl 192.168.2.38      # checking frontend -> backend
    frontend calling backend. response from backend: message from backend
    /app # curl localhost:8101    # can check directly because inside the container
    frontend calling backend. response from backend: message from backend
    /app # exit

Test with a another pod. Note: We'll launch this pod in the default namespace.

    $ kubectl run tester -it --rm --restart=Never -n default --image=ubuntu sh
    # apt-get update ; apt-get install curl -y
    # curl web.ruby-frontend
    frontend calling backend. response from backend: message from backend
    #

Note, you will not be able to curl web.ruby-frontend from the tester container because it's in the default namespace.

    # curl web.ruby-backend
    curl: (28) Failed to connect to web.ruby-backend port 80: Connection timed out
    #

Remember, only the ruby-backend and ruby-frontend namespace can reach the backend pods. This is because the ruby-backend app has a NetworkPolicy with these rules.

## Delete and Cleanup

Note, if you're still testing the ruby-frontend app, don't delete this app yet until. The frontend app talks to this backend app.

    $ kubes delete -y
    Compiled  .kubes/resources files to .kubes/output
    => kubectl delete -f .kubes/output/web/deployment.yaml
    deployment.apps "web" deleted
    => kubectl delete -f .kubes/output/web/service.yaml
    service "web" deleted
    => kubectl delete -f .kubes/output/shared/network_policy.yaml
    networkpolicy.networking.k8s.io "web" deleted
    => kubectl delete -f .kubes/output/shared/service_account.yaml
    serviceaccount "ruby-backend" deleted
    => kubectl delete -f .kubes/output/shared/namespace.yaml
    namespace "ruby-backend" deleted
    $

You can also clean up the backend app also now. Cd into it's project folder and run `kubes delete -y`.

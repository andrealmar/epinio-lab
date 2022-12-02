# Epinio Lab

Epinio is a next generation PaaS built with existing CNCF projects (LinkerD, Cert-Manager, Traefik and others) which aims to provide a great developer experience. Epinio is opinionated and runs on Kubernetes to take you from code to URL in one step. 

If you want to learn more about it go to https://epinio.io. 

## Rationale behind the creation of Epinio

Kubernetes is becoming the de-facto standard for container orchestration. Developers may want to use Kubernetes for all the benefits it provides or may have to do so because that's what their Ops team has chosen. Whatever the case, using Kubernetes is not simple. It has a steep learning curve and doing it right is a full time job. Developers should spend their time working on their applications, not doing operations.

Epinio is adding the needed abstractions and intelligence to allow Developers to use Kubernetes as a PaaS (Platform as a Service).

## Installation

In order to use Epinio in a quickly manner, we are going to use some tools to make our lives easier being: 

#### Docker

https://www.docker.com

#### Helm

https://helm.sh/

#### kubectl

https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/

#### k3d

https://k3d.io/v5.4.6/

#### Epinio CLI

https://github.com/epinio/epinio/releases/tag/v1.5.1

> P.S - I've got some problems when installing CLI via Homebrew (`brew install epinio`) so I installed using the latest release binary on Github (see link above).

### K3D Cluster

Assuming you already have Docker installed in your machine (I'm using a Macbook M1) let's install our local Kubernetes cluster. We are going to use [K3D](https://k3d.io/v5.4.6/) for our local Kubernetes cluster. Follow the instructions on K3D page to install it in your machine. 

Open your Terminal and type:

```shell
k3d cluster create epinio -p '80:80@loadbalancer' -p '443:443@loadbalancer'
```

to create the local Kubernetes cluster with K3D. 

Epinio has to connect to pods inside the cluster. The default installation uses the internal Docker IP for this. If Docker is running in a VM, e.g. with Docker Desktop for Mac (which is my case), that IP will not be reachable.

We have to create the cluster in a way, that the internal port 80 (where the traefik ingress controller is listening on) is exposed on the host system.

That's why we used `-p '80:80@loadbalancer' -p '443:443@loadbalancer'` which means we are mapping the ingress port 80 to `localhost:80` and we are also mapping port 443 to `localhost:443`.

The port-mapping construct `80:80@loadbalancer` means:

- “map port `80` from the host to port 80 on the container which matches the nodefilter `loadbalancer`“

- The loadbalancer nodefilter matches only the serverlb that’s deployed in front of a cluster’s server nodes

- All ports exposed on the `serverlb` will be proxied to the same ports on all server nodes in the cluster

### cert-manager

After the cluster is created, we need to install `cert-manager` there:

Create the `cert-manager` namespace:

```shell
kubectl create ns cert-manager
```

Add the Jetstack Helm repo: 

```shell
helm repo add jetstack https://charts.jetstack.io
```

Do a Helm update:

```shell
helm repo update
```

Install cert-manager:

```shell
helm install cert-manager --namespace cert-manager jetstack/cert-manager \
		--set installCRDs=true \
		--set extraArgs[0]=--enable-certificate-owner-ref=true
```

### Traefik

Before deploying Epinio, we must have an ingress controller available in our cluster. NGINX and Traefik provide two popular options. 

Ingresses let you expose your applications using URLs instead of raw hostnames and ports. Epinio requires your apps to be deployed with a URL, so it won’t work without an Ingress Controller. New deployments automatically generate a URL, but you can manually assign one instead. 

Most popular single-node Kubernetes distributions such as K3s,minikube and Rancher Desktop come with one either built-in or as a bundled add-on. 

Luckily, we are using K3D which deploys Traefik as the default Ingress Controller.

To see the Traefik deployment rollout status in our local K3D cluster, run:

```shell
kubectl rollout status deployment traefik -n kube-system
```

A message like this one below will appear in your Terminal:

```shell
deployment "traefik" successfully rolled out
```

which means we are good to proceed installing Epinio as our next step. 

### Epinio

Add the Epinio Helm repo: 

```shell
helm repo add epinio https://epinio.github.io/helm-charts
```

Install Epinio via Helm: 

```shell
helm install epinio -n epinio --create-namespace --version 1.4.0 epinio/epinio \
		--set global.domain=127.0.0.1.sslip.io \
		--set api.users[0].role=admin \
		--set api.users[0].username=admin \
		--set api.users[0].password=password \
		--set api.users[1].role=user \
		--set api.users[1].username=epinio \
		--set api.users[1].password=password
```

Note that we are using the flag `--set global.domain=127.0.0.1.sslip.io` which means we are using a "Magic DNS". Our choice for "Magic DNS" was https://sslip.io/. 

A working system domain is needed for Epinio to work. We are just trying out Epinio, so we can use a "magic" domain to get running faster. By "magic" we mean a domain that will always resolve to the IP address that is part of the domain itself.

After the installation is done you should see:

```shell
NAME: epinio
LAST DEPLOYED: Tue Nov 30 22:03:57 2022
NAMESPACE: epinio
STATUS: deployed
REVISION: 1
NOTES:
To interact with your Epinio installation download the latest epinio binary from https://github.com/epinio/epinio/releases/latest.

Login to the cluster with any of

    `epinio login -u admin https://epinio.127.0.0.1.sslip.io`
    `epinio login -u epinio https://epinio.127.0.0.1.sslip.io`

or go to the dashboard at: https://epinio.127.0.0.1.sslip.io

If you didn't specify a password the default one is `password`.

For more information about Epinio, feel free to checkout https://epinio.io/ and https://docs.epinio.io/.
```

You can now go to your browser and visit the URL https://epinio.127.0.0.1.sslip.io

![Epinio Welcome Page](/img/1.png "Epinio Welcome Page")

Type your credentials (admin/password) and Login into Epinio UI

![Epinio UI](/img/2.png "Epinio UI")

Let's create an application via Epinio UI. Click on the Create button on top right corner. 

Select __Git URL__ as the _Source Type_, https://githuc.com/andrealmar/pokemon-api as _URL_ and __master__ as a _Branch_ as you can see in the picture below:

![Epinio App Creation](/img/3.png "Epinio App Creation")

In the next step, type a name for the application (_pokemon-api_ in my case) and select the number of desired instances (_1_ in my case):

![Epinio App Creation](/img/4.png "Epinio App Creation")

Next step is to select a _Service_ to bind to our application but in this case it is not necessary so we can just click "Create" in the bottom right corner. 

![Epinio App Creation](/img/5.png "Epinio App Creation")

You will see Epinio doing 4 steps in the next screen (Create App, Fetch, Build and Deploy):

![Epinio App Creation](/img/6.png "Epinio App Creation")

Also you can see the application logs using Epinio UI:

![Epinio App Creation](/img/7.png "Epinio App Creation")

We can see that Epinio created a new pod for our application: 

```shell
➜  kubectl get pods -n workspace
NAME                                                              READY   STATUS    RESTARTS   AGE
rpokemon-api-61c8fa3c31e6c14570da1778b9e5bfa9b63fb669-56cdvvqlm   1/1     Running   0          5s
```

If you go back to applications in Epinio UI you will see our _pokemon-api_ successfully deployed:

![Epinio App Creation](/img/8.png "Epinio App Creation")

#### Let's test our app:

Go to https://pokemon-api.127.0.0.1.sslip.io and you will be greated with:

```shell
Welcome to POKEMON API from 10.42.0.48
```

> P.S - 10.42.0.48 is the IP of our Pod

Now go to the URL https://pokemon-api.127.0.0.1.sslip.io/pikachu and you will see:

```shell
quu..__
 $$$b  `---.__
  "$$b        `--.                          ___.---uuudP
   `$$b           `.__.------.__     __.---'      $$$$"              .
     "$b          -'            `-.-'            $$$"              .'|
       ".                                       d$"             _.'  |
         `.   /                              ..."             .'     |
           `./                           ..::-'            _.'       |
            /                         .:::-'            .-'         .'
           :                          ::''\          _.'            |
          .' .-.             .-.           `.      .'               |
          : /'$$|           .@"$\           `.   .'              _.-'
         .'|$u$$|          |$$,$$|           |  <            _.-'
         | `:$$:'          :$$$$$:           `.  `.       .-'
         :                  `"--'             |    `-.     \
        :##.       ==             .###.       `.      `.    `\
        |##:                      :###:        |        >     >
        |#'     `..'`..'          `###'        x:      /     /
         \                                   xXX|     /    ./
          \                                xXXX'|    /   ./
          /`-.                                  `.  /   /
         :    `-  ...........,                   | /  .'
         |         ``:::::::'       .            |<    `.
         |             ```          |           x| \ `.:``.
         |                         .'    /'   xXX|  `:`M`M':.
         |    |                    ;    /:' xXXX'|  -'MMMMM:'
         `.  .'                   :    /:'       |-'MMMM.-'
          |  |                   .'   /'        .'MMM.-'
          `'`'                   :  ,'          |MMM<
            |                     `'            |tbap\
             \                                  :MM.-'
              \                 |              .''
               \.               `.            /
                /     .:::::::.. :           /
               |     .:::::::::::`.         /
               |   .:::------------\       /
              /   .''               >::'  /
              `',:                 :    .'
                                   `:.:' Tim Park
 

```

You can also try with the following endpoints: 

- https://pokemon-api.127.0.0.1.sslip.io/bulbasaur
- https://pokemon-api.127.0.0.1.sslip.io/squirtle
- https://pokemon-api.127.0.0.1.sslip.io/charmander



> P.S - In order to make the _pokemon-api_ work with Epinio I had to add a `Procfile`. You can check that in the repo. 

## Rails App (with DB backend)

We are going to install the Rails App inside the `/example-rails` folder.

Make sure the Ruby version in your machine matches the Ruby version of the application (2.7.5):

```shell
brew install rbenv
rbenv install 2.7.5
rbenv global 2.7.5
```

Add the Bitnami Helm repo:
```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
```

Update Helm repos:

```shell
helm repo update
```

Installing the PostgreSQL database in our K3D cluster:

```shell
helm install postgres bitnami/postgresql --set volumePermissions.enabled=true,postgresqlUsername=myuser,postgresqlPassword=mypassword,postgresqlDatabase=production
```

Output:

```shell
NAME: postgres
LAST DEPLOYED: Thu Dec  1 17:11:43 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: postgresql
CHART VERSION: 12.1.2
APP VERSION: 15.1.0

** Please be patient while the chart is being deployed **

PostgreSQL can be accessed via port 5432 on the following DNS names from within your cluster:

    postgres-postgresql.default.svc.cluster.local - Read/Write connection

To get the password for "postgres" run:

    export POSTGRES_PASSWORD=$(kubectl get secret --namespace default postgres-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)

To connect to your database run the following command:

    kubectl run postgres-postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:15.1.0-debian-11-r0 --env="PGPASSWORD=$POSTGRES_PASSWORD" \
      --command -- psql --host postgres-postgresql -U postgres -d postgres -p 5432

    > NOTE: If you access the container using bash, make sure that you execute "/opt/bitnami/scripts/postgresql/entrypoint.sh /bin/bash" in order to avoid the error "psql: local user with ID 1001} does not exist"

To connect to your database from outside the cluster execute the following commands:

    kubectl port-forward --namespace default svc/postgres-postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432

```

which means the installation was successful. 

You can double check with: 

```shell
➜  kubectl get pods 
NAME                    READY   STATUS    RESTARTS   AGE
postgres-postgresql-0   1/1     Running   0          50s
```

Now let's create our Rails App in Epinio using the Epinio CLI: 

```shell
epinio apps create rails-example
```

Output: 

```shell
🚢  Create application
Namespace: workspace
Application: rails-example

⚠️  Epinio server version (v1.4.0) doesn't match the client version (v1.5.0)

⚠️  Update the client manually or run `epinio client-sync`

✔️  Ok

```

Check if the Rails app was created with:

```shell
epinio apps list
```

Output:

```shell
🚢  Listing applications
Namespace: workspace

⚠️  Epinio server version (v1.4.0) doesn't match the client version (v1.5.0)

⚠️  Update the client manually or run `epinio client-sync`

✔️  Epinio Applications:
|     NAME      |            CREATED            | STATUS |             ROUTES             |           CONFIGURATIONS            | STATUS DETAILS |
|---------------|-------------------------------|--------|--------------------------------|-------------------------------------|----------------|
| pokemon-api   | 2022-12-01 15:51:55 -0300 -03 | 1/1    | pokemon-api.127.0.0.1.sslip.io |                                     |                |
| rails-example | 2022-12-01 17:13:49 -0300 -03 | n/a    | n/a                            |                                     |                |
| wordpress     | 2022-12-01 10:01:40 -0300 -03 | 1/1    | wordpress.127.0.0.1.sslip.io   | xcca9aa0f19a036fb6389474a7be0-mysql |                |

```

You can see that `rails-example` app was created. 

Create a service with Epinio to allow the application to access the database:

```shell
epinio service create postgresql-dev mydb 
```

Bind the Service to the Application

```shell
epinio service bind mydb rails-example
```

Create an environment variable for RAILS_MASTER_KEY:

```shell
epinio apps env set rails-example RAILS_MASTER_KEY $(cat config/master.key)
```

Now push the rails application with Epinio CLI:

```shell
epinio push -n rails-example
```

> P.S - I had to modify the Ruby version on `Gemfile` of `rails-example` app because Paketo Buildpacks are only compatible with the following Ruby versions:

```shell
Supported versions are: [2.7.6, 2.7.7, 3.0.4, 3.0.5, 3.1.2, 3.1.3] 
```

It will take a while to spin up the Rails application so grab a coffee, sit back and relax. 

If everything works as expected, the application deployment should finish soon and the command should print the url of your app (something like https://rails-example.127.0.0.1.sslip.io). 

Visit that in your browser and see the greeting message.

## DOCKER & LOCAL ENVIRONMENT

### LOCAL

- had to move python to dependencies and change the version to match the version on my machine
- had to install postgres in order to have the pg_config utility required by some dependency when doing the `conda env create` command
- had to install Rust compiler
- Tried to install miniconda via Homebrew but I wasn't successful because of Architecture problems (ARM vs x86_64) so I had to install miniconda by downloading the `pkg` file compatible with the python version I was using (installing via homebrew didn't work) 
- had to type this command: `pip install psycopg2-binary --force-reinstall --no-cache-dir` to force reinstall of `psycopg2-binary` in my local machine and fix the `libpq5 not found` error. 

### DOCKER

- I had to include the postgresql package in the main Dockerfile
- I've installed `RUN pip install psycopg2 --force-reinstall --no-cache-dir` on orders service Dockerfile
- on docker-compose I changed the postgres container version to 11 (it was getting the latest version)
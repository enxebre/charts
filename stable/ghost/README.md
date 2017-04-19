# Ghost

[Ghost](https://ghost.org/) is one of the most versatile open source content management systems on the market.

## TL;DR;

```console
$ helm install stable/ghost
```

## Cluster Deployment

Tested with:

- Kubernetes v1.6.1_coreos.0 deployed in aws with Kubeform.

- bitnami/ghost:0.11.7-r1

- Helm Client: &version.Version{SemVer:"v2.3.0", GitCommit:"d83c245fc324117885ed83afc90ac74afed271b4", GitTreeState:"clean"}

- Kubectl Client Version: version.Info{Major:"1", Minor:"6", GitVersion:"v1.6.1", GitCommit:"b0b7a323cc5a4a2019b2e9520c21c7830b7f708e", GitTreeState:"clean", BuildDate:"2017-04-03T20:44:38Z", GoVersion:"go1.7.5", Compiler:"gc", Platform:"darwin/amd64"}

```helm dependency update```

```helm install -f custom-values.yaml ./```

```helm upgrade -f custom-values.yaml --set replicas=3 loitering-ibex ./```

### Creating a self contained Helm Repo
If you have restricted access on your environment you can make your chart available on a local helm server on demand by running a Docker file like this:

```
FROM alpine:3.4
RUN apk --update add ca-certificates
RUN mkdir /usr/chart
COPY . /usr/chart
RUN /usr/chart/helm_installer.sh
WORKDIR /usr/chart
RUN helm lint
RUN helm package --save=false .
CMD helm serve --address 0.0.0.0:8879 --repo-path .
```
https://github.com/enxebre/prometheus-operated-chart

### Autoscaling
We use autoscaling to test how we can scale our app.

```kubectl autoscale deployment nasal-pig-ghost --min=2 --max=5 --cpu-percent=2```

Generate load:

```kubectl run -i --tty service-test --image=busybox /bin/sh```

and run

```while true; do wget -q -O- http://nasal-pig-ghost.52.215.24.82.nip.io/; done```

Then you can scale the service-test deployment.

### Problem

We want to deploy the Ghost Chart with Helm and scale it horizontally.

When deploying the Ghost chart a few things happen.

- Within the container at build time:

```nami unpack```

https://github.com/bitnami/bitnami-docker-ghost/blob/0.11.7-r1/0/Dockerfile#L17

https://github.com/bitnami/minideb-extras/blob/e99ac61664fc12e6cbdcc1005dac7a1f497dd73e/jessie/rootfs/usr/local/bin/bitnami-pkg

- At runtime:

```nami initialize```

https://github.com/bitnami/bitnami-docker-ghost/blob/0.11.7-r1/0/rootfs/app-entrypoint.sh#L9

https://github.com/bitnami/minideb-extras/blob/e99ac61664fc12e6cbdcc1005dac7a1f497dd73e/jessie/rootfs/opt/bitnami/base/helpers

Which will run a post intallation hook unless a given folder specified in bitnami.json is initialized:

https://downloads.bitnami.com/files/stacksmith/ghost-0.11.7-0-linux-x64-debian-8.tar.gz#

```json
    "persistDir": {
      "description": "Directory to backup application folders",
      "value": "/bitnami/ghost"
    },
```

And it's not overridable at the env level https://github.com/bitnami/bitnami-docker-ghost/blob/0.11.7-r1/0/rootfs/ghost-inputs.json

So the hook will try to do a full bootstrap, creating a database and config file and syslinking the "content" folder every time you run the container which we don't want:

```javascript
  if (!volumeFunctions.isInitialized($app.persistDir)) {
    const domain = _.isEmpty($app.host) ? networkFunctions.getMachineIp() : $app.host;
    databaseHandler.checkConnection();
    databaseHandler.createDatabaseForApp($app.name);
    $app.info('==> Creating database...');
    $file.copy('config.example.js', confFile,
                {owner: {username: $app.systemUser, group: $app.systemGroup}});
    $file.substitute(confFile, /host:\s*'127.0.0.1'/, 'host: \'0.0.0.0\'');
    $file.substitute(confFile, /client:\s*'sqlite3'/, 'client: \'mysql\'');
    $file.substitute(confFile, /filename:\s*path\.join\(__dirname,\s*'\/content\/data\/ghost.*'\)/,
                     `host     : '${databaseHandler.connection.host}',
                port     : '${databaseHandler.connection.port}',
                user     : '${databaseHandler.appDatabase.user}',
                password : '${databaseHandler.appDatabase.password}',
                database : '${databaseHandler.appDatabase.name}',
                charset  : 'utf8'`);
    $app.helpers.configureHost(domain);
    $app.helpers.createAdminUser();
    $app.helpers.configureSMTP($app.smtpHost, $app.smtpUser, $app.smtpPassword, $app.smtpService);
    volumeFunctions.prepareDataToPersist($app.dataToPersist);
  } else {
    volumeFunctions.restorePersistedData($app.dataToPersist);
  }
 ```

### Potential solutions

- To change the run command so the entrypoint does not execute ```nami_initialize ghost```

	https://github.com/bitnami/bitnami-docker-ghost/blob/0.11.7-r1/0/rootfs/app-entrypoint.sh#L8

	So run this straight away:

	```"command": "{{$app.installdir}}/node_modules/pm2/bin/pm2 start -p {{$app.installdir}}/tmp/pids/ghost.pid --merge-logs -l {{$app.logFile}}  index.js"```

	*:x: Nami complains as it needs to finish the package installation. Side effects.*

- To change the entrypoint so it doesn't run ```nami_initialize ghost```

	*:x: We'd be skipping nami life cycle management.*

- To share the given folder across all the Ghost instances in the cluster so they see the same data.

	*:x: We don't want unnecessary state in services. Antipatern. Constrained by cloud, regions, underneath storage solution, etc.*

- Assume we know the config ahead of time. Move it into a Kubernetes configMap. Prepopulate the ghost folder with a .initialized file so no bootstrap is required. The content should be moved out to a different layer.

	https://github.com/TryGhost/Ghost/wiki/Using-a-custom-storage-module

	:white_check_mark:

```yaml
persistence:
  enabled: false

serviceType: ClusterIP

# Expose the service to an ingressController that will load balance across all the pods.
ingressDomain: 52.215.24.82.nip.io

# This will create a new user with permissions for the "blog" database. This parameters will be used by the configMap.
mariadb:
  mariadbRootPassword: rootUser0
  mariadbUser: testUser
  mariadbPassword: testUser0
  mariadbDatabase: blog
```

We inject the config via volume plugin:

```yaml
  volumeMounts:
  - name: config-volume
    mountPath: /bitnami/ghost/config.js
    subPath: config.js
  - name: config-volume
    mountPath: /bitnami/ghost/.initialized
    subPath: .initialized.js          
volumes:
- name: config-volume
  configMap:
    name: {{ template "fullname" . }}        
    items:
      - key: config.js
        path: config.js
      - key: .initialized
        path: .initialized
```

See also https://medium.com/capgemini-engineering/releasing-backward-incompatible-changes-kubernetes-jenkins-plugin-prometheus-operator-helm-self-6263ca61a1b1

### Demo

[![Helm Ghost Autoscaling](https://img.youtube.com/vi/eYQoGSmjIEA/0.jpg)](https://www.youtube.com/watch?v=eYQoGSmjIEA)


## Introduction

This chart bootstraps a [Ghost](https://github.com/bitnami/bitnami-docker-ghost) deployment on a [Kubernetes](http://kubernetes.io) cluster using the [Helm](https://helm.sh) package manager.

It also packages the [Bitnami MariaDB chart](https://github.com/kubernetes/charts/tree/master/stable/mariadb) which is required for bootstrapping a MariaDB deployment for the database requirements of the Ghost application.

## Prerequisites

- Kubernetes 1.4+ with Beta APIs enabled
- PV provisioner support in the underlying infrastructure

## Installing the Chart

To install the chart with the release name `my-release`:

```console
$ helm install --name my-release stable/ghost
```

The command deploys Ghost on the Kubernetes cluster in the default configuration. The [configuration](#configuration) section lists the parameters that can be configured during installation.

> **Tip**: List all releases using `helm list`

## Uninstalling the Chart

To uninstall/delete the `my-release` deployment:

```console
$ helm delete my-release
```

The command removes all the Kubernetes components associated with the chart and deletes the release.

## Configuration

The following tables lists the configurable parameters of the Ghost chart and their default values.

| Parameter                         | Description                                           | Default                                                   |
| --------------------------------- | ----------------------------------------------------- | --------------------------------------------------------- |
| `image`                           | Ghost image                                           | `bitnami/ghost:{VERSION}`                                 |
| `imagePullPolicy`                 | Image pull policy                                     | `Always` if `image` tag is `latest`, else `IfNotPresent`  |
| `ghostHost`                       | Ghost host to create application URLs                 | `nil`                                                     |
| `ghostPort`                       | Ghost port to create application URLs along with host | `80`                                                      |
| `ghostLoadBalancerIP`             | `loadBalancerIP` for the Ghost Service                | `nil`                                                     |
| `ghostUsername`                   | User of the application                               | `user`                                                    |
| `ghostPassword`                   | Application password                                  | Randomly generated                                        |
| `ghostEmail`                      | Admin email                                           | `user@example.com`                                        |
| `ghostBlogTitle`                  | Ghost Blog name                                       | `User's Blog`                                             |
| `mariadb.mariadbRootPassword`     | MariaDB admin password                                | `nil`                                                     |
| `serviceType`                     | Kubernetes Service type                               | `LoadBalancer`                                            |
| `persistence.enabled`             | Enable persistence using PVC                          | `true`                                                    |
| `persistence.storageClass`        | PVC Storage Class for Ghost volume                    | `nil` (uses alpha storage annotation) |
| `persistence.accessMode`          | PVC Access Mode for Ghost volume                      | `ReadWriteOnce`                                           |
| `persistence.size`                | PVC Storage Request for Ghost volume                  | `8Gi`                                                     |
| `resources`                       | CPU/Memory resource requests/limits                   | Memory: `512Mi`, CPU: `300m`                              |

The above parameters map to the env variables defined in [bitnami/ghost](http://github.com/bitnami/bitnami-docker-ghost). For more information please refer to the [bitnami/ghost](http://github.com/bitnami/bitnami-docker-ghost) image documentation.

> **Note**:
>
> For the Ghost application function correctly, you should specify the `ghostHost` parameter to specify the FQDN (recommended) or the public IP address of the Ghost service.
>
> Optionally, you can specify the `ghostLoadBalancerIP` parameter to assign a reserved IP address to the Ghost service of the chart. However please note that this feature is only available on a few cloud providers (f.e. GKE).
>
> To reserve a public IP address on GKE:
>
> ```bash
> $ gcloud compute addresses create ghost-public-ip
> ```
>
> The reserved IP address can be associated to the Ghost service by specifying it as the value of the `ghostLoadBalancerIP` parameter while installing the chart.

Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`. For example,

```console
$ helm install --name my-release \
  --set ghostUsername=admin,ghostPassword=password,mariadb.mariadbRootPassword=secretpassword \
    stable/ghost
```

The above command sets the Ghost administrator account username and password to `admin` and `password` respectively. Additionally it sets the MariaDB `root` user password to `secretpassword`.

Alternatively, a YAML file that specifies the values for the above parameters can be provided while installing the chart. For example,

```console
$ helm install --name my-release -f values.yaml stable/ghost
```

> **Tip**: You can use the default [values.yaml](values.yaml)

## Persistence

The [Bitnami Ghost](https://github.com/bitnami/bitnami-docker-ghost) image stores the Ghost data and configurations at the `/bitnami/ghost` and `/bitnami/apache` paths of the container.

Persistent Volume Claims are used to keep the data across deployments. This is known to work in GCE, AWS, and minikube.
See the [Configuration](#configuration) section to configure the PVC or to disable persistence.

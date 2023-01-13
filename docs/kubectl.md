# Install Percona XtraDB Cluster using kubectl

[kubectl](https://kubernetes.io/docs/tasks/tools/) is Kubernetes command-line tool. The kubectl allows users to run commands against Kubernetes clusters. Users can use kubectl to deploy applications, manage cluster resources, and check logs.

## Pre-requisites

The following tools are used in this guide and therefore should be preinstalled:

1. **Git** is distributed version control software. You can install it following the [official installation instructions](https://github.com/git-guides/install-git).

2. **kubectl**  to manage and deploy applications on Kubernetes. Install
it [following the official installation instructions](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

The following steps are needed to run the Operator and Percona XtraDB Cluster on
your k8s cluster:

1. Deploy the operator with the following command:

    ```default
    $ kubectl apply -f https://raw.githubusercontent.com/percona/percona-xtradb-cluster-operator/v{{ release }}/deploy/bundle.yaml
    ```

    As the result you will have operator pod up and running. To check the state of this pod run the following command.
    `kubectl get pods` output should look like this:

    ``` {.text .no-copy}
    NAME                                            READY   STATUS    RESTARTS   AGE
    percona-xtradb-cluster-operator-d99c748-sqddq   1/1     Running   0          1m
    ```

2. Deploy Percona XtraDB Cluster:

    ```default
    $ kubectl apply -f https://raw.githubusercontent.com/percona/percona-xtradb-cluster-operator/v{{ release }}/deploy/cr-minimal.yaml
    ```

    !!! note

        This deploys one Percona XtraDB Cluster node and one HAProxy node. The
        [deploy/cr-minimal.yaml](https://raw.githubusercontent.com/percona/percona-xtradb-cluster-operator/v{{ release }}/deploy/cr-minimal.yaml) is for minimal non-production deployment.
        For more configuration options please see
        [deploy/cr.yaml](https://raw.githubusercontent.com/percona/percona-xtradb-cluster-operator/v{{ release }}/deploy/cr.yaml) and [Custom Resource Options](operator.md).

    Creation process will take some time. The process is over when both
    operator and replica set pod have reached their Running status.
    `kubectl get pods` output should look like this:

    ``` {.text .no-copy}
    NAME                                            READY   STATUS    RESTARTS   AGE
    percona-xtradb-cluster-operator-d99c748-sqddq   1/1     Running   0          6m
    minimal-cluster-pxc-0                           3/3     Running   0          2m
    minimal-cluster-haproxy-0                       2/2     Running   0          2m
    ```

3. During previous steps, the Operator has generated several [secrets](https://kubernetes.io/docs/concepts/configuration/secret/), including the
    password for the `root` user, which you will definitely need to access the
    cluster. Use `kubectl get secrets minimal-cluster-secrets -o jsonpath='{.data.root}`
    command to get root password which should look as follows:
    ``` {.text .no-copy}
    ZDdLaS1QUkgkWik+IU5AbA==
    ```
    
    Use `kubectl get secrets` to see the list of Secrets objects (by
    default Secrets object you are interested in has `minimal-cluster-secrets` name).

    Here the actual password is base64-encoded, and
    `kubectl get secrets minimal-cluster-secrets -o jsonpath='{.data.root}' | base64 -d` will bring it back to a human-readable form.

4. Check connectivity to a newly created cluster.

    First of all, run a MySQL client container and connect its console output to your
    terminal (running it may require some time to deploy the correspondent Pod).
    The following command will do this, naming the new Pod `percona-client`:

    ``` {.bash data-prompt="$" }
    $ kubectl run -i --rm --tty percona-client --image=percona:8.0 --restart=Never -- bash -il
    ```

    Now run `mysql` tool in the percona-client command shell using the password
    obtained from the secret:

    ``` {.bash data-prompt="$" }
    $ mysql -h minimal-cluster-haproxy -uroot -proot_password
    ```

    This command will connect you to the MySQL server.
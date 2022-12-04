# Install Percona XtraDB Cluster on OpenShift

Percona Operator for Percona XtrabDB Cluster is a [Red Hat Certified Operator](https://connect.redhat.com/en/partner-with-us/red-hat-openshift-certification). This means that Percona Operator is portable across hybrid clouds and fully supports the Red Hat OpenShift lifecycle.

Installing Percona XtraDB Cluster on OpenShift includes two steps:


* Installing the Percona Operator for MySQL,


* Install Percona XtraDB Cluster using the Operator.

## Install the Operator

You can install Percona Operator for MySQL on OpenShift using the [Red Hat Marketplace](https://marketplace.redhat.com) web interface or using the command line interface.

### Install the Operator via the Red Hat Marketplace


1. login to the Red Hat Marketplace and register your cluster [following the official instructions](https://marketplace.redhat.com/en-us/workspace/clusters/add/register).

2. Go to the [Percona Operator for MySQL](https://marketplace.redhat.com/en-us/products/percona-kubernetes-operator-for-percona-server-for-xtradb-cluster) page and click the Free trial button:

    ![image](img/marketplace-operator-page.png)

    Here you can “start trial” of the Operator for 0.0 USD.

3. When finished, chose `Workspace->Software` in the system menu on the top and choose the Operator:

    ![image](img/marketplace-operator-install.png)

    Click the `Install Operator` button.

### Install the Operator via the command-line interface

1. Clone the percona-xtradb-cluster-operator repository:

    ``` {.bash data-prompt="$" }
    $ git clone -b v{{ release }} https://github.com/percona/percona-xtradb-cluster-operator
    $ cd percona-xtradb-cluster-operator
    ```

!!! note

    It is crucial to specify the right branch with the -b
    option while cloning the code on this step. Please be careful.

2. Now Custom Resource Definition for Percona XtraDB Cluster should be created
    from the `deploy/crd.yaml` file. Custom Resource Definition extends the
    standard set of resources which Kubernetes “knows” about with the new
    items (in our case ones which are the core of the operator).

    This step should be done only once; it does not need to be repeated
    with the next Operator deployments, etc.

    ``` {.bash data-prompt="$" }
    $ oc apply -f deploy/crd.yaml
    ```

    !!! note

        Setting Custom Resource Definition requires your user to
        have cluster-admin role privileges.

    If you want to manage your Percona XtraDB Cluster with a non-privileged user,
    necessary permissions can be granted by applying the next clusterrole:

    ``` {.bash data-prompt="$" }
    $ oc create clusterrole pxc-admin --verb="*" --resource=perconaxtradbclusters.pxc.percona.com,perconaxtradbclusters.pxc.percona.com/status,perconaxtradbclusterbackups.pxc.percona.com,perconaxtradbclusterbackups.pxc.percona.com/status,perconaxtradbclusterrestores.pxc.percona.com,perconaxtradbclusterrestores.pxc.percona.com/status
    $ oc adm policy add-cluster-role-to-user pxc-admin <some-user>
    ```

    If you have a [cert-manager](https://docs.cert-manager.io/en/release-0.8/getting-started/install/openshift.html) installed, then you have to execute two more commands to be able to     manage certificates with a non-privileged user:

    ``` {.bash data-prompt="$" }
    $ oc create clusterrole cert-admin --verb="*" --resource=issuers.certmanager.k8s.io,certificates.certmanager.k8s.io
    $ oc adm policy add-cluster-role-to-user cert-admin <some-user>
    ```

3. The next thing to do is to create a new `pxc` project:

    ``` {.bash data-prompt="$" }
    $ oc new-project pxc
    ```

4. Now RBAC (role-based access control) for Percona XtraDB Cluster should be set
    up from the `deploy/rbac.yaml` file. Briefly speaking, role-based access is
    based on specifically defined roles and actions corresponding to
    them, allowed to be done on specific Kubernetes resources (details
    about users and roles can be found in [OpenShift documentation](https://docs.openshift.com/enterprise/3.0/architecture/additional_concepts/authorization.html)).

    ``` {.bash data-prompt="$" }
    $ oc apply -f deploy/rbac.yaml
    ```

    Finally, it’s time to start the operator within OpenShift:

    ``` {.bash data-prompt="$" }
    $ oc apply -f deploy/operator.yaml
    ```

## Install Percona XtraDB Cluster

1. Now that’s time to add the Percona XtraDB Cluster Users secrets to OpenShift.
    They should be placed in the data section of the `deploy/secrets.yaml`
    file as logins and plaintext passwords for the user accounts
    (see [Kubernetes documentation](https://kubernetes.io/docs/concepts/configuration/secret/)
    for details).

    After editing is finished, users secrets should be created using the
    following command:

    ``` {.bash data-prompt="$" }
    $ oc create -f deploy/secrets.yaml
    ```

    More details about secrets can be found in [Users](users.md#users).

2. Now certificates should be generated. By default, the Operator generates
    certificates automatically, and no actions are required at this step. Still,
    you can generate and apply your own certificates as secrets according
    to the [TLS instructions](TLS.md#tls).

3. After the operator is started and user secrets are added, Percona
    XtraDB Cluster can be created at any time with the following command:

    ``` {.bash data-prompt="$" }
    $ oc apply -f deploy/cr.yaml
    ```

    Creation process will take some time. The process is over when both
    operator and replica set pod have reached their Running status:

    ``` {.text .no-copy}
    NAME                                               READY   STATUS    RESTARTS   AGE
    cluster1-haproxy-0                                 2/2     Running   0          6m17s
    cluster1-haproxy-1                                 2/2     Running   0          4m59s
    cluster1-haproxy-2                                 2/2     Running   0          4m36s
    cluster1-pxc-0                                     3/3     Running   0          6m17s
    cluster1-pxc-1                                     3/3     Running   0          5m3s
    cluster1-pxc-2                                     3/3     Running   0          3m56s
    percona-xtradb-cluster-operator-79966668bd-rswbk   1/1     Running   0          9m54s
    ```

4. Check connectivity to newly created cluster. Run a container with MySQL 
    monitor and connect its console output to your
    terminal. The following command will do this, naming the new Pod
    `percona-client`:

    ``` {.bash data-prompt="$" }
    $ oc run -i --rm --tty percona-client --image=percona:8.0 --restart=Never -- bash -il
    percona-client:/$ mysql -h cluster1-haproxy -uroot -proot_password
    ```

    This command will connect you to the MySQL monitor.

    ``` {.text .no-copy}
    mysql: [Warning] Using a password on the command line interface can be insecure.
    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 1976
    Server version: 8.0.19-10 Percona XtraDB Cluster (GPL), Release rel10, Revision 727f180, WSREP version 26.4.3

    Copyright (c) 2009-2020 Percona LLC and/or its affiliates
    Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
    ```

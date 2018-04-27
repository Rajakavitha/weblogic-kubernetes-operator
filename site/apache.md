
# Load Balancing with Apache Web Server

This page describes how to setup and start a Apache Web Server for load balancing inside a Kubernets cluster. The configuration and startup can either be automatic when you create a domain using the WebLogic Operator's `create-weblogic-domain.sh` script, or manually if you have an existing WebLogic domain configuration.

## Build Docker Image for Apache Web Server

You need to prepare the Docker image for Apache Web Server that enbeds Oracle WebLogic Server Proxy Plugin.

  1. Download and build the Docker image for the Apache Web Server with 12.2.1.3.0 Oracle WebLogic Server Proxy Plugin.  See the instructions in

[https://github.com/oracle/docker-images/tree/master/OracleWebLogic/samples/12213-webtier-apache).

  2. tag your Docker image to `store/oracle/apache:12.2.1.3` using `docker tag` command.

```

docker tag 12213-apache:latest store/oracle/apache:12.2.1.

```

More information about the Apache plugin can be found at: [https://docs.oracle.com/middleware/1213/webtier/develop-plugin/apache.htm#PLGWL395)

Once you have access to the Docker image of the Apache Web Server, you can go ahead follow the instructions below to setup and start Kubernetes artifacts for Apache Web Server.


## When use Apache load balancer with a WebLogic domain created with the WebLogic Operator

Please refer to [Creating a domain using the WebLogic Operator](creating-domain.md) for how to create a domain with the WebLogic Operator.

You need to configure Apache Web Server as your load balancer for a WebLogic domain by setting the `loadBalancer` option to `APACHE` in `create-weblogic-domain-inputs.yaml` (as shown below) when running the `create-weblogic-domain.sh` script to create a domain. 

```

# Load balancer to deploy.  Supported values are: VOYAGER, TRAEFIK, APACHE, NONE

loadBalancer: APACHE

```

The `create-weblogic-domain.sh` script installs the Apache Web Server with Oracle WebLogic Server Proxy Plugin into the Kubernetes *cluster*  in the same namespace of the *domain*.

The Apache Web Server will expose a `NodePort` that allows access to the load balancer from the outside of the Kubernetes cluster.  The port is controlled by the following setting in `create-weblogic-domain-inputs.yaml` file:

```

# Load balancer web port

loadBalancerWebPort: 30305

```

The user can access an application from utsidie of the Kubernetes cluster via http://<host>:30305/<application-url>.

### Use the default plugin WL module configuration

By default, the Apache Docker image supports a simple WebLogic server proxy plugin configuration for a single WebLogic domain with an admin server and a cluster. The `create-weblogic-domain.sh` script automatically customizes the default behavior based on your domain configuration. The default setting only supports the type of load balancing that uses the root path ("/"). You can further customize the root path of the load balancer withloadBalancerAppPrepath property in the `create-weblogic-domain-inputs.yaml` file.

```

# Load balancer app prepath

loadBalancerAppPrepath: /weblogic

```

The user can then access an application from utsidie of the Kubernetes cluster via `http://<host>:30305/weblogic/<application-url>,` and the admin can then access the admin console via `http://<host>:30305/console`.

The generated Kubernetes yaml files look like the following given the domainUID "domain1".

`weblogic-domain-apache.yaml` for Apache web server deployment

```

--

apiVersion: v1

kind: ServiceAccount

metadata:

  name: domain1-apache-webtier

  namespace: default

  labels:

    weblogic.domainUID: domain1

    weblogic.domainName: base_domain

    app: apache-webtier

---

kind: Deployment

apiVersion: extensions/v1beta1

metadata:

  name: domain1-apache-webtier

  namespace: default

  labels:

    weblogic.domainUID: domain1

    weblogic.domainName: base_domain

    app: apache-webtier

spec:

  replicas: 1

  selector:

    matchLabels:

      weblogic.domainUID: domain1

      weblogic.domainName: base_domain

      app: apache-webtier

  template:

    metadata:

      labels:

        weblogic.domainUID: domain1

        weblogic.domainName: base_domain

        app: apache-webtier

    spec:

      serviceAccountName: domain1-apache-webtier

      terminationGracePeriodSeconds: 60

      # volumes:

      # - name: "domain1-apache-webtier"

      #   hostPath:

      #     path: %LOAD_BALANCER_VOLUME_PATH%

      containers:

      - name: domain1-apache-webtier

        image: store/oracle/apache:12.2.1.3

        imagePullPolicy: Never

        # volumeMounts:

        # - name: "domain1-apache-webtier"

        #   mountPath: "/config"

        env:

          - name: WEBLOGIC_CLUSTER

            value: 'domain1-cluster-cluster-1:8001'

          - name: LOCATION

            value: '/weblogic'

          - name: WEBLOGIC_HOST

            value: 'domain1-admin-server'

          - name: WEBLOGIC_PORT

            value: '7001'

        readinessProbe:

          tcpSocket:

            port: 80

          failureThreshold: 1

          initialDelaySeconds: 10

          periodSeconds: 10

          successThreshold: 1

          timeoutSeconds: 2

        livenessProbe:

          tcpSocket:

            port: 80

          failureThreshold: 3

          initialDelaySeconds: 10

          periodSeconds: 10

          successThreshold: 1

          timeoutSeconds: 2

---

apiVersion: v1

kind: Service

metadata:

  name: domain1-apache-webtier

  namespace: default

  labels:

    weblogic.domainUID: domain1

    weblogic.domainName: base_domain

spec:

  type: NodePort

  selector:

    weblogic.domainUID: domain1

    weblogic.domainName: base_domain

    app: apache-webtier

  ports:

    - port: 80

      nodePort: 30305

      name: rest-https

```

`weblogic-domain-apache-security.yaml` for associated RBAC roles and role bindings

```

---

kind: ClusterRole

apiVersion: rbac.authorization.k8s.io/v1beta1

metadata:

  name: domain1-apache-webtier

  labels:

    weblogic.domainUID: domain1

    weblogic.domainName: base_domain

rules:

  - apiGroups:

      - ""

    resources:

      - pods

      - services

      - endpoints

      - secrets

    verbs:

      - get

      - list

      - watch

  - apiGroups:

      - extensions

    resources:

      - ingresses

    verbs:

      - get

      - list

      - watch

---

kind: ClusterRoleBinding

apiVersion: rbac.authorization.k8s.io/v1beta1

metadata:

  name: domain1-apache-webtier

  labels:

    weblogic.domainUID: domain1

    weblogic.domainName: base_domain

roleRef:

  apiGroup: rbac.authorization.k8s.io

  kind: ClusterRole

  name: domain1-apache-webtier

subjects:

- kind: ServiceAccount

  name: domain1-apache-webtier

  namespace: default

```


Here are examples of the Kubernetes artifacts created by the WebLogic Operator:


```


AMESPACE             NAME                            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE

default               deploy/domain1-apache-webtier   1         1         1            1           4h



default               po/domain1-apache-webtier-7f7cf44dcf-pqzn7          1/1       Running   1          4h

PORT(S)           AGE

default               svc/domain1-apache-webtier                      NodePort    10.96.36.137     <none>        80:30305/TCP      4h


NAME                            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE

bash-4.2$ kubectl get all |grep apache

deploy/domain1-apache-webtier   1         1         1            1           2h


rs/domain1-apache-webtier-7b8b789797   1         1         1         2h


deploy/domain1-apache-webtier   1         1         1            1           2h


rs/domain1-apache-webtier-7b8b789797   1         1         1         2h



po/domain1-apache-webtier-7b8b789797-lm45z   1/1       Running   0          2h


svc/domain1-apache-webtier                      NodePort    10.111.114.67    <none>        80:30305/TCP      2h



bash-4.2$ kubectl get clusterroles |grep apache

domain1-apache-webtier                                                 2h



bash-4.2$ kubectl get clusterrolebindings |grep apache

domain1-apache-webtier                                    2h

```


### Use your own plugin WL module configuration


Optionally you can fine turn the behavior of the Apache plugin by providing your own Apache plugin configuration for `create-weblogic-domain.sh` script to use. You put your custom_mod_wl_apache.conf file in a local directory, say `/scratch/apache/config` , and specify this location in the `create-weblogic-domain-inputs.yaml` file as follows.

```

# Docker volume path for APACHE

# By default, the VolumePath is empty, which will cause the volume mount be disabled

loadBalancerVolumePath: /scratch/apache/config

```

Once the loadBalancerVolumePath is specified, the `create-weblogic-domain.sh` script will use the custom_mod_wl_apache.config file in `/scratch/apache/config` directory to replace what is in the Docker image.

The generated yaml files will look similar except that the lines that start with "#" will be un-commented like the following.

```

      volumes:

      - name: "domain1-apache-webtier"

        hostPath:

          path: /scratch/apache/config 

      containers:

      - name: domain1-apache-webtier

        image: store/oracle/apache:12.2.1.3

        imagePullPolicy: Never

        volumeMounts:

        - name: "domain1-apache-webtier"

          mountPath: "/config"

```



## When use Apache load balancer with a manually created WebLogic Domain

If your WebLogic domain is not created by the WebLogic Operator, you need to manually create and start all Kubernetes' artifacts for Apache Web Server.


  1. Create your own custom_mod_wl_apache.conf file, and put it in a local dir, say `<apache-conf-dir>`. See the instructions in [https://docs.oracle.com/middleware/1213/webtier/develop-plugin/apache.htm#PLGWL395).

  2. Create the Apache deployment yaml file. See the example above. Note that you need to use the **volumes** and **volumeMounts** to mount `<apache-config-dir>` in to `/config` directory inside the pod that runs Apache web tier. Note that the Apache Web Server needs to be in the same Kubernetes namespace as the WebLogic domains that it needs to access.

  3. Create a RBAC yaml file. See the example above

Note that you can choose to run one Apache Web Server to load balance to multiple domains/clusters inside the same Kubernetes cluster as long as the Apache Web Server and the domains are all in the same namespace.








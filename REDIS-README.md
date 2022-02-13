# Get Redis Helm Chart

**_* Add chart repository locally -Setup Once- *_**
helm repo add bitnami https://charts.bitnami.com/bitnami

**_* Download a chart specified version from helm repository and unpacks it in local directory *_**
helm pull bitnami/redis --version 16.3.0 --untar

**\_\__ OPTION-1 _\_\_**

# Update your helm values and deploy using kubectl apply

Reference the helm values `redis-test-helm-values` for all the required changes.

_IMPORTANT_

**_*set installCRDs=true *_**

```

```

# Render chart locally and validate add --namespace flag to generate the chart for particular namespace

`helm template redis --namespace redis --values redis-test-helm-values.yaml --output-dir ./manifests/test/ redis`

_IMPORTANT BEFORE APPLY_

**\_\__ VOLUME ACCESS _\_\_**

Because Bitnami charts run all the containers as runAsNonRoot we need to apply a monkey patch to have mariadb statefullset run on our linode.
We need to add an INIT-CONTAINER that is going to go give the correct access to the Volume created.
The chart values file creates the Init-Container for us BUT make sure to update the volumePermission section of the chart and set it to true
and that the both the podSecurityContext and containerSecurityContext are configured with the correct fsGroup

# Set volume-permissions Init-Container defintion

```
## 'volumePermissions' init container parameters
## Changes the owner and group of the persistent volume mount point to runAsUser:fsGroup values
##   based on the *podSecurityContext/*containerSecurityContext parameters
##
volumePermissions:
  ## @param volumePermissions.enabled Enable init container that changes the owner/group of the PV mount point to `runAsUser:fsGroup`
  ##
  enabled: true
```

# If the configuration is done correclty generating the template would create the initContainer for the MariaDB statefullset

`manifests/test/redis/templates/master/statefulset.yaml`
This will start an init container using the image defined and peformed the required actions as root user

1- Create the required directory `mkdir -p "/data"`
2- Grant the correct access right to the user 1001 `chown -R "1001:1001" "/data"` to the required directory

```
      initContainers:
        - name: volume-permissions
          image: docker.io/bitnami/bitnami-shell:10-debian-10-r329
          imagePullPolicy: "IfNotPresent"
          command:
            - /bin/bash
            - -ec
            - |
              chown -R 1001:1001 /data
          securityContext:
            runAsUser: 0
          resources:
            limits: {}
            requests: {}
          volumeMounts:
            - name: redis-data
              mountPath: /data
              subPath:
```

**\_\__ Note that the issue that prompeted the change in the securityContext was still observed even after runing with root \*_\***
**\_\_\_ The resolution was to update the file permission for the moodledata directory to chmod 777 \_\_\_**

```
Fatal error: $CFG->dataroot is not writable, admin has to fix directory permissions! Exiting
ls -la
drwxr-xr-x  5 root root  4096 Feb  6 02:50 .
drwxr-x---  3 root root  4096 Feb  9 02:28 ..
drwx------  2 root root 16384 Feb  6 02:50 lost+found
drwxrwxr-x 59 root root  4096 Feb  8 16:39 moodle
drwxrwxrwx 12 root root  4096 Feb  8 16:06 moodledata  #Gave Full Permission to this directory chmod 777

```

**_*Create a ConfigMap for the redis deployment to mount the ''' file has a volume. Gives control over Server Configs*_**

Need to have access over the Redis server configs in the php.ini file located in the container @ ``
Create a configmap with all the ` ` configs and mount it by adding it to the deployment of Redis.

```

```

# Update Redis deployment template generated with the config map mounted as a volume

```

```

# Apply all the rendered yaml files using kubectl apply

`k apply -Rf manifests/test/redis -n redis`

# Un-install - Delete all the rendered yaml files

`k delete -Rf manifests/test/redis -n redis`

**\_\__ OPTION-2 _\_\_**

# Use the Helm upgrade --install and pass the values instead.

`helm upgrade --install redis bitnami/redis --namespace redis `

# Un-install using helm uninstal - Delete all the objects created

`helm uninstall redis -n redis`

_POST INSTALL CONFIGURATION_

#
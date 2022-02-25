# Deploy Redis in LKE

### Get Redis Helm Chart

**Add chart repository locally -Setup Once**
helm repo add bitnami https://charts.bitnami.com/bitnami

**Download a chart specified version from helm repository and unpacks it in local directory**
helm pull bitnami/redis --version 16.3.0 --untar

## OPTION-1

### Update your helm values and deploy using kubectl apply

Reference the helm values `redis-test-helm-values` for all the required changes for Test

Reference `redis-prod-helm-values` for Prod

### Render chart locally and validate add `--namespace` flag to generate the chart for particular namespace

_Test_
`helm template redis --namespace redis --values redis-test-helm-values.yaml --output-dir ./manifests/test/ redis`

_Prod_
`helm template redis --namespace redis --values redis-prod-helm-values.yaml --output-dir ./manifests/prod/ redis`

_IMPORTANT BEFORE APPLY_

**VOLUME ACCESS**

Because Bitnami charts run all the containers as runAsNonRoot we need to add an INIT-CONTAINER that is going to go give the correct access to the Volume created.
The chart values file creates the Init-Container for us BUT make sure to update the volumePermission section of the chart and set it to true
Also ensure both the podSecurityContext and containerSecurityContext are configured with the correct fsGroup

### Set volume-permissions Init-Container defintion

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

If the configuration is done correclty generating the template would create the initContainer for the Redis statefullset
`manifests/test/redis/templates/master/statefulset.yaml`.
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

** Note certain apps still require change of permissions using chmod.
** Example: chmod -R 777 /(directory name) or chmod 755 -R /(directory name)

## OPTIONAL

**Create a ConfigMap for the redis deployment to mount the ''' file has a volume. Gives control over Server Configs**

### Apply all the rendered definitions yaml files using kubectl apply

_Test_
`k apply -Rf manifests/test/redis -n redis`

_Prod_
`k apply -Rf manifests/prod/redis -n redis`

### Uninstall \ Delete all the rendered yaml files

_Test_
`k delete -Rf manifests/test/redis -n redis`

_Prod_
`k delete -Rf manifests/prod/redis -n redis`

## OPTION-2

### Use `helm upgrade --install` and pass the values file.

_Test_
`helm upgrade --install redis bitnami/redis --namespace redis`

_Prod_
`helm upgrade --install redis bitnami/redis --namespace redis`

### Uninstall using `helm uninstal`

`helm uninstall redis -n redis`

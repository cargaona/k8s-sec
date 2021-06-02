# RBAC
RBAC es un acrónimo para "Role Based Access Control". Kubernetes implementa esta forma de conceder accesos a usuarios y aplicaciones. 
Podríamos decir que RBAC concede accesos luego de contestar la siguiente pregunta: ¿Puede tal persona ejecutar tal acción sobre de terminado objeto?. De esto inferimos 3 partes necesarias. Un sujeto (Tranquilos, no es análisis sintáctico esto.). Una acción (Les juro que esto no es un predicado verbal simple). Y un objeto (No, no es un objeto directo. Tampoco hay voz pasiva o activa aunque parezca.)

Veamoslo del siguiente modo: 
Yo tengo un gato que se llama Pochoclo. Él va a ser mi sujeto a quien voy a darle permisos.
Formulemos la pregunta de la siguiente manera:
¿Puede Pochoclo dormir en el sillón?

Para resolver esto usando RBAC necesitamos generar roles que le dejen a Pochoclo jugar con el peluche.

Para eso podemos crear un rol llamado "Mascota Malcriada", con los permisos "Dormir en el sillón" y "Subirse arriba de la mesa". 

El rol "Mascota Malcríada" existe en todos los departamentos de mi edificio, es decir es definido globalmente, pero Pochoclo sólo tiene ese rol en mi departamento, no en el de todos mis vecinos, por lo que podemos decir que asume ese rol localmente. 

Por otra parte, puedo crear un rol muy específico que exista sólo en mi casa. Es decir, localmente. Este rol va a ser "Mascota Peleadora" y va a tener el permiso para pelear con mi otra gata: Aceituna. De este modo el rol es definido localmente, sólo existe en mi casa, y es asumido también localmente por Pochoclo. 

Resulta que Pochoclo es un gato muy confianzudo, y le gusta ir a tomar agua a los departamentos de mis vecinos. Para esto podemos definir un role "Mascota Confianzuda" con los permisos "Tomar agua". Este role va a ser global, existe en todos los departamentos del edificio, pero esta vez Pochoclo va a asumir el rol globalmente, ya que no sólo va a tomar agua en mi casa: Va a tomar agua en todos los departamentos.

Con esto que contamos vemos que hay 3 comportamientos posibles. 
- Rol Global. Asumido Globalmente.
- Rol Global. Asumido Localmente.   
- Rol Local. Asumido Localmente. 

Kubernetes nos da herramientas para lograr esto, a través de los Roles (roles locales) y  ClusterRoles (roles globales, a través de todo el clúster). Y también con los RoleBindings (Relacion de Rol o ClusterRole a un recurso Local) y ClusterRoleBindings (Relación de Rol o ClusterRole a un recurso Global. )

Los Roles y ClusterRoles son utilizados para autenticar usuarios y darle determinados permisos sobre el cluster. 
Kubernetes no tiene una API para crear usuarios, pero sí sabe autenticarlos. El manejo de usuarios está mayormente resuelto por los Cloud Providers.  

## Crear un user en Kubernetes
Vamos a crear un user utilizando certificados.

- Generamos keys usando OpenSSL:
```bash
openssl genrsa -out pochoclo.key 2048
```
- Generate a Client Sign Request (CSR)
```bash
openssl req -new -key pochoclo.key -out pochoclo.csr -subj “/CN=pochoclo/O=group1”
```
- Generate the certificate (CRT)
```bash
openssl x509 -req -in pochoclo.csr -CA ~/.minikube/ca.crt -CAkey ~/.minikube/ca.key -CAcreateserial -out pochoclo.crt -days 500
```

- Set a context entry for the user in the kubeconfig file. (~/.kubeconfig)
```bash
kubectl config set-context pochoclo-minikube --cluster=minikube --user=pochoclo
```

- Set an entry for the user in kubeconfig.
```bash
kubectl config set-credentials pochoclo --client-certificate=pochoclo.crt --client-key=pochoclo.key
```

## Creación de un Role
Para darle permisos al usuario que creamos vamos a generar un manifests para un "Role". 
```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pods-reader
rules:
- apiGroups: [""] 
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

En este caso, el role que creamos es local, por eso es un Role y no un ClusterRole. Como es local, tenemos que definir a qué namespace corresponde. En nuestro caso, responde al namespace 'default'.
Dentro de 'metadata' tenemos el nombre del Role, que nos va a servir luego para relacionarlo con el user, así como cuando nombrábamos los roles 'Mascota Confianzuda' o 'Mascota Malcríada'.

Más abajo vemos las 'Rules'.
Dentro de las reglas, tenemos 'apiGroups': Dentro de la ApiServer a qué grupos de endpoints podemos hacer pedidos. Más info sobre los API Groups existentes [acá](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.21/#-strong-api-groups-strong-). En este caso al dejarlas como "" estamos dando acceso a todos los API Groups.

También tenemos los 'resources'. En este caso le estamos indicando que este Role otorgará permisos a los recursos u objetos 'pod'.

Por último tenemos los 'verbs'. Las acciones que quien asuma este Role podrá ejecutar sobre los recursos declarados.

Recuerdan que era 'sujeto, acción, recurso'?. Vamos a hacer que Pochoclo, pueda hacer 'get', 'watch' y 'list' sobre 'pods'.

## Relacionando Rol con User

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: pochoclo
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pods-reader 
  apiGroup: rbac.authorization.k8s.io
```

Este objeto, 'RoleBinding', nos sirve para relacionar Roles o ClusterRoles de manera Local. Es decir, sólo en un namespace. 
Dentro de 'Subjects' tenemos 'Kind'. Lo que va allí responde a la pregunta ¿A qué cosa querés relacionarle este rol?. Y las posibles respuestas son Group, User o ServiceAccount. #TODO link a docu.
También está 'name', que es el nombre del 'Kind' elegido. En nuestro caso es 'pochoclo' porque ese ese el nombre de usuario.
Más abajo tenemos 'RoleRef' donde ponemos la información del Role que vamos a relacionar, en nuestro caso, al usuario Pochoclo. Completamos con los datos correspondientes y estamos en condiciones de aplicar. 

```bash
» cd ./../roles-pochoclo/local-local
» k apply -f ./
role.rbac.authorization.k8s.io/pods-reader created
rolebinding.rbac.authorization.k8s.io/read-pods created
```
## Prueba del Usuario y sus permisos.
Lo primero que vamos a hacer es pasarnos al 'context' del user _pochoclo_. Para eso necesitamos ejecutar el siguiente comando.
```bash
» kubectl config use-context minikube-pochoclo
Switched to context "minikube-pochoclo".
```

Podemos probar que nuestros permisos funcionen:
```bash
» kubectl get pods
NAME          READY   STATUS    RESTARTS   AGE
dnsutils      1/1     Running   2          155m
dnsutils-sa   1/1     Running   1          80m
```

Podemos probar alguna acción para la que no tenemos permisos. 

```
 » k get secrets
Error from server (Forbidden): secrets is forbidden: User "pochoclo" cannot list resource "secrets" in API group "" in the namespace "default"
```

Tal como esperábamos no deberíamos tener acceso a los secretos en este namespace. 

Como es un Role local, por lo tanto asumido localmente, no podíamos listar pods en otro Namespace. 

``` bash
» kubectl get pods -n kube-system
Error from server (Forbidden): pods is forbidden: User "pochoclo" cannot list resource "pods" in API group "" in the namespace "kube-system"
```

#TODO escribir global-local y global-global.

## ServiceAccount
Los objetos ServiceAccount son últiles para cuando queremos dar acceso a aplicaciones. 
Así como vimos que se pueden hacer "RoleBindings" y "ClusterRoleBindings" entre Roles, ClusterRoles y Users también se puede hacer lo mismo con ServiceAccounts.


## En qué casos una aplicación quiere acceder a algún recurso de Kubernetes?
Hay varios ejemplos de aplicaciones de "Infraestructura" que pueden querer tener datos sobre nuestro Cluster. Ejemplo: Aplicaciones de monitoreo que quieran tener datos acerca de la cantidad de Nodos que tiene nuestro cluster, nombres, uso, y cantidad de pods/deployments. Otras aplicaciones pueden tener permisos no sólo de lectura sobre un cluster. Como por ejemplo ClusterAutoscaler, que se encarga de aumentar o disminuir la cantidad de nodos que tiene nuestro cluster en base a la disponibilidad de memoria o cpu. Esta aplicacion necesita tener permisos para crear nuevos nodos, también para eliminarlos en caso de que no tengan uso. Aplicaciones como ArgoCD o Flux, populares aplicaciones de CI/CD para Kubernetes necesitan permisos de escritura, generalmente en todos los namespaces, para poder deployar lo que el desarrollador _pushea_ a un repositorio de Git.  

## Default ServiceAccount
Por defecto, cualquier cosa que deployemos dentro del cluster va a usar un ServiceAccount por defecto a menos que le indiquemos lo contrario. Este Service Account tiene permisos acotados. Se recomienda generar un ServiceAccount por cada aplicación o grupo de aplicaciones para poder auditar de una manera más sencilla qué aplicación está usando qué recurso y también para poder aplicar el principio de mínimos privilegios. 

Qué permisos tiene el default-serviceAccount?

- 
-


## Cómo funciona el serviceAccount? Kubernetes ApiServer.

Como vimos al principio, la manera de interactuar con Kubernetes es a través de su API y las aplicaciones no son la excepción. Podemos intentar hacer un request a la Kubernetes API para ver qué nos devuelve. 

Para esta prueba vamos a utilizar una imagen de docker conocida como 'dns-utils' que nos permite troubleshootear problemas de DNS y Networking dentro de un cluster. Podés instalarla del siguiente modo:
```bash
kubectl apply -f https://k8s.io/examples/admin/dns/dnsutils.yaml
```

Una vez _deployado_ el dns-utils podemos ejecutar el siguiente comando para ver qué nos devuelve la Kubernetes API.

_Si te genera dudas cómo es que sólo "kubernetes" resuelve a la ApiServer, podés correr el siguiente comando y entender a dónde está resolviendo:_
```bash
$ k exec -it dnsutils -- host kubernetes                                                                

kubernetes.default.svc.cluster.local has address 10.96.0.1
```bash
k exec -it dnsutils -- curl https://kubernetes/api/v1 --insecure  | jq .
```

Debería devolvernos algo así:

```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/api/v1\"",
  "reason": "Forbidden",
  "details": {},
  "code": 403
}
```

Parece que llegamos a la ApiServer sin problemas pero no tenemos permisos, por eso obtuvimos un 403. Ahora es donde entra en juego el #service-account-token.

## ServiceAccount-Token
Podemos listar los ServiceAccounts que existen en nuestro default namespace y vamos a ver que sólo existe el default. 
```bash
» k get serviceaccounts                                                                                 <<<
NAME      SECRETS   AGE
default   1         20h
```

Si indagamos un poco más podemos describirlo, a ver qué tenemos. 

```bash
 » k describe serviceaccounts                                                                            <<<
Name:                default
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   default-token-xbddk
Tokens:              default-token-xbddk
Events:              <none>
```

Parece que tenemos un "default-token-xbddk". No parece ser un misterio lo que hay allí adentro. Vamos a leerlo.

```bash
k get secrets/default-token-xbddk --template={{.data.token}} | base64 -d                              <<<
eyJhbGciOiJSUzI1NiIsImtpZCI6Ing2dGJWczNOSnk5VFA0N2Y5b3cySE56YjM3OGZJcU1wR0YyTWZIbTZQTk0ifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlZmF1bHQtdG9rZW4teGJkZGsiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGVmYXVsdCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjY4YTVmNDYzLWIwZWQtNGFjYS1hYTZjLTAwYzFkNDFlMjQzMiIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.WWrBS3olgviRrQj0uPwKMJOxWkLLdGkWJ6DTEdXBGBCpQgzA4gtV3jxxUWIWncdxnLY4dyQeEXqGBNnbUBhnPWfQOQbX1sJx2i9__nc1Gq7EITm8yHAtF-rpXHYMhbOrhb1tDyxiIuyvkEwcOxYR_Oas9asZ6jnxFJ02cogfDdlCfyKUz9r4FZ9rpHWtFf2T4-40BrcwsOu7FUShf624985aQFOoNHTzlYSnS6ENdTz96Eid5jUtT1pDVxWjzYL3Fi7cr5ZQmHXAosVe_8Kf4nKhdJqckyEtQCOwO6L16kP6Oo_zslWP0x5c5oxU74gh_5hJKzjHuiI_PJN40k9LAg
```

Con este comando nos traemos el "secret-token". Si todo marcha bien podríamos volver a hacer el request que hicimos antes a la API pero esta vez debería ser exitoso.

## ApiServer request 2. 

Guardamos el token obtenido anteriormente en una variable llamada TOKEN. 

```bash
k exec -it dnsutils -- curl -H "Authorization: Bearer $TOKEN" https://kubernetes/api/v1 --insecure
```

Parece que ahora tuvimos éxito. 


¿Qué pasa si queremos ver qué pods hay? 
``` bash
» k exec -it dnsutils -- curl -H "Authorization: Bearer $TOKEN" https://kubernetes/api/v1/pods --insecure  | jq .     

{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "pods is forbidden: User \"system:serviceaccount:default:default\" cannot list resource \"pods\" in API group \"\" at the cluster scope",
  "reason": "Forbidden",
  "details": {
    "kind": "pods"
  },
  "code": 403
}
```
Nos encontramos con una de las limitaciones del default ServiceAccount. Generemos uno con los permisos necesarios y veamos si podemos listar los pods.

## Generando un ServiceAccount y _Bindeandolo_ con un Role. 

Podemos dar una mirada a los archivos que están en ./service-account/service-account-binding.
```bash
.
├── rolebinding.yaml
├── role.yaml
└── service-account.yaml
```

En role tenemos los permisos (De hecho son los mismos que usamos para generar los permisos del user "Pochoclo", pero los traemos acá para hacerlo más claro).
En rolebinding tenemos el binding entre el role y el serviceaccount. 
En service-account.yaml tenemos la definición del ServiceAccount. 


Una vez que aplicamos estos manifests, un nuevo service-token debería haberse generado, pero esta vez para el "demo-service-account".

Listamos los secretos para estar seguros de que existe.
```bash
» k get secrets
NAME                               TYPE                                  DATA   AGE
default-token-xbddk                kubernetes.io/service-account-token   3      21h
demo-service-account-token-gwb97   kubernetes.io/service-account-token   3      3m25s
```

Vemos que ahora también está el "demo-service-account-token". Obtengamos ese token y volvamos a ejecutar el request a ver si ahora somos capaces de listar los pods.

```bash
» k get secrets/demo-service-account-token-gwb97 --template={{.data.token}} | base64 -d
eyJhbGciOiJSUzI1NiIsImtpZCI6Ing2dGJWczNOSnk5VFA0N2Y5b3cySE56YjM3OGZJcU1wR0YyTWZIbTZQTk0ifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlbW8tc2VydmljZS1hY2NvdW50LXRva2VuLWd3Yjk3Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImRlbW8tc2VydmljZS1hY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiMjM5ZjZiM2ItZjQ0Yi00NGRjLWJjNjAtODlhY2Y1NjAyZTUwIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmRlZmF1bHQ6ZGVtby1zZXJ2aWNlLWFjY291bnQifQ.TLI51AAJBiR2VfwZfIh6NXPHmRdPLvMqXwnnaPZM8xz3S6A65meCm5ZT3uZFIMmDYXvdliRs_2rZb6dAeSXAIKt9uUB9bbG66ZxeTGJzksYKR6sksj-g-MoYZKC5dqOEOXPCpSB13psKV6e2-AGW82sj0HNeKeREh_6vPYSvGQEHmp6E6JqVTk69Xln5AMDJzWIndXHbputuA3Bc9VIilYaA0kUVxGmt4869xHGQduLc22eO50sQfBPK4JD48AZuO9iYVuCMrmUzhgCCeKvAcEzEDXiMCXaBxNz3djSONRcIo4CtzJHhF3m9Exy35hAX0ZfnFZPicVUW91TajP4xNA
```
Copiamos el token a la variable TOKEN.

```bash
 k exec -it dnsutils -- curl -H "Authorization: Bearer $TOKEN" https://kubernetes/api/v1/namespaces/default/pods --insecure  | jq .
```

```json
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "55011"
  },
  "items": [
    {
      "metadata": {
        "name": "dnsutils",
        "namespace": "default",
        "uid": "b91022f0-bf19-4713-b910-8ceb0cf7798e",
        "resourceVersion": "54852",
        "creationTimestamp": "2021-05-25T16:00:35Z",
        "annotations": {
          "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"name\":\"dnsutils\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"sleep\",\"3600\"],\"image\":\"gcr.io/kubernetes-e2e-test-images/dnsutils:1.3\",\"imagePullPolicy\":\"IfNotPresent\",\"name\":\"dnsutils\"}],\"restartPolicy\":\"Always\"}}\n"
        },
        "managedFields": [
          {
            "manager": "kubectl-client-side-apply",
            "operation": "Update",
            "apiVersion": "v1",
            "time": "2021-05-25T16:00:35Z",
            "fieldsType": "FieldsV1",
            "fieldsV1": {
              "f:metadata": {
                "f:annotations": {
                  ".": {},
                  "f:kubectl.kubernetes.io/last-applied-configuration": {}
                }
              },
              "f:spec": {
                "f:containers": {
                  "k:{\"name\":\"dnsutils\"}": {
                    ".": {},
                    "f:command": {},
                    "f:image": {},
                    "f:imagePullPolicy": {},
                    "f:name": {},
                    "f:resources": {},
                    "f:terminationMessagePath": {},
                    "f:terminationMessagePolicy": {}
                  }
                },
                "f:dnsPolicy": {},
                "f:enableServiceLinks": {},
                "f:restartPolicy": {},
                "f:schedulerName": {},
                "f:securityContext": {},
                "f:terminationGracePeriodSeconds": {}
              }
            }
          },
          {
            "manager": "kubelet",
            "operation": "Update",
            "apiVersion": "v1",
            "time": "2021-05-25T17:00:44Z",
            "fieldsType": "FieldsV1",
            "fieldsV1": {
              "f:status": {
                "f:conditions": {
                  "k:{\"type\":\"ContainersReady\"}": {
                    ".": {},
                    "f:lastProbeTime": {},
                    "f:lastTransitionTime": {},
                    "f:status": {},
                    "f:type": {}
                  },
                  "k:{\"type\":\"Initialized\"}": {
                    ".": {},
                    "f:lastProbeTime": {},
                    "f:lastTransitionTime": {},
                    "f:status": {},
                    "f:type": {}
                  },
                  "k:{\"type\":\"Ready\"}": {
                    ".": {},
                    "f:lastProbeTime": {},
                    "f:lastTransitionTime": {},
                    "f:status": {},
                    "f:type": {}
                  }
                },
                "f:containerStatuses": {},
                "f:hostIP": {},
                "f:phase": {},
                "f:podIP": {},
                "f:podIPs": {
                  ".": {},
                  "k:{\"ip\":\"172.17.0.3\"}": {
                    ".": {},
                    "f:ip": {}
                  }
                },
                "f:startTime": {}
              }
            }
          }
        ]
      },
      "spec": {
        "volumes": [
          {
            "name": "default-token-xbddk",
            "secret": {
              "secretName": "default-token-xbddk",
              "defaultMode": 420
            }
          }
        ],
        "containers": [
          {
            "name": "dnsutils",
            "image": "gcr.io/kubernetes-e2e-test-images/dnsutils:1.3",
            "command": [
              "sleep",
              "3600"
            ],
            "resources": {},
            "volumeMounts": [
              {
                "name": "default-token-xbddk",
                "readOnly": true,
                "mountPath": "/var/run/secrets/kubernetes.io/serviceaccount"
              }
            ],
            "terminationMessagePath": "/dev/termination-log",
            "terminationMessagePolicy": "File",
            "imagePullPolicy": "IfNotPresent"
          }
        ],
        "restartPolicy": "Always",
        "terminationGracePeriodSeconds": 30,
        "dnsPolicy": "ClusterFirst",
        "serviceAccountName": "default",
        "serviceAccount": "default",
        "nodeName": "minikube",
        "securityContext": {},
        "schedulerName": "default-scheduler",
        "tolerations": [
          {
            "key": "node.kubernetes.io/not-ready",
            "operator": "Exists",
            "effect": "NoExecute",
            "tolerationSeconds": 300
          },
          {
            "key": "node.kubernetes.io/unreachable",
            "operator": "Exists",
            "effect": "NoExecute",
            "tolerationSeconds": 300
          }
        ],
        "priority": 0,
        "enableServiceLinks": true,
        "preemptionPolicy": "PreemptLowerPriority"
      },
      "status": {
        "phase": "Running",
        "conditions": [
          {
            "type": "Initialized",
            "status": "True",
            "lastProbeTime": null,
            "lastTransitionTime": "2021-05-25T16:00:35Z"
          },
          {
            "type": "Ready",
            "status": "True",
            "lastProbeTime": null,
            "lastTransitionTime": "2021-05-25T17:00:44Z"
          },
          {
            "type": "ContainersReady",
            "status": "True",
            "lastProbeTime": null,
            "lastTransitionTime": "2021-05-25T17:00:44Z"
          },
          {
            "type": "PodScheduled",
            "status": "True",
            "lastProbeTime": null,
            "lastTransitionTime": "2021-05-25T16:00:35Z"
          }
        ],
        "hostIP": "192.168.49.2",
        "podIP": "172.17.0.3",
        "podIPs": [
          {
            "ip": "172.17.0.3"
          }
        ],
        "startTime": "2021-05-25T16:00:35Z",
        "containerStatuses": [
          {
            "name": "dnsutils",
            "state": {
              "running": {
                "startedAt": "2021-05-25T17:00:43Z"
              }
            },
            "lastState": {
              "terminated": {
                "exitCode": 0,
                "reason": "Completed",
                "startedAt": "2021-05-25T16:00:42Z",
                "finishedAt": "2021-05-25T17:00:42Z",
                "containerID": "docker://565f1cc0e2f6d438badd398987aff477c9259eabbcce1e570c7eb24f9a309b73"
              }
            },
            "ready": true,
            "restartCount": 1,
            "image": "gcr.io/kubernetes-e2e-test-images/dnsutils:1.3",
            "imageID": "docker-pullable://gcr.io/kubernetes-e2e-test-images/dnsutils@sha256:b31bcf7ef4420ce7108e7fc10b6c00343b21257c945eec94c21598e72a8f2de0",
            "containerID": "docker://eed0bdc592b6ae16891c9f5390d8f1e45f87f6374d6ea59e5b7c7fe70f8e74e1",
            "started": true
          }
        ],
        "qosClass": "BestEffort"
      }
    }
  ]
}
```

Parece que esta vez tuvimos éxito. Ya vimos como crear un ServiceAccount y usar el token generado para utilizar la Kubernetes ApiServer. Pero hasta ahora estuvimos tomando ese token a mano y ejecutando requests por nosotros mismos. ¿Cómo obtienen estos permisos las aplicaciones? 


## Binding ServiceAccounts a Pods o Deployments.
Vamos a _bindear_ nuestro Service Account creado anteriormente "demo-service-account" a un nuevo deployment de dns-utils. 
El yaml está en ./dns-utils/deployment.yaml. Podemos ver que dentro de _Specs_ tenemos un campo de ServiceAccountName. Ahí es donde debemos introducir el nombre del ServiceAccount a relacionar y a partir de ahí nuestro deployment tendrá los permisos asignados a ese ServiceAccount.

Todo perfecto hasta acá... ¿Pero cómo es utilizado el token dentro del container? Revisemos un poco.

Veamos qué info tenemos acerca del pod.

```bash
» k describe pods dnsutils-sa

Name:         dnsutils-sa
Namespace:    default
Priority:     0
Node:         minikube/192.168.49.2
Start Time:   Tue, 25 May 2021 14:15:36 -0300
Labels:       <none>
Annotations:  <none>
Status:       Running
IP:           172.17.0.4
IPs:
  IP:  172.17.0.4
Containers:
  dnsutils-sa:
    Container ID:  docker://8691c36db0eb4d36fd61b656ae82d1a7845754c489e7a86aebac8ebe116fb706
    Image:         gcr.io/kubernetes-e2e-test-images/dnsutils:1.3
    Image ID:      docker-pullable://gcr.io/kubernetes-e2e-test-images/dnsutils@sha256:b31bcf7ef4420ce7108e7fc10b6c00343b21257c945eec94c21598e72a8f2de0
    Port:          <none>
    Host Port:     <none>
    Command:
      sleep
      3600
    State:          Running
      Started:      Tue, 25 May 2021 14:15:37 -0300
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from demo-service-account-token-gwb97 (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  demo-service-account-token-gwb97:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  demo-service-account-token-gwb97
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  5m20s  default-scheduler  Successfully assigned default/dnsutils-sa to minikube
  Normal  Pulled     5m19s  kubelet            Container image "gcr.io/kubernetes-e2e-test-images/dnsutils:1.3" already present on machine
  Normal  Created    5m19s  kubelet            Created container dnsutils-sa
  Normal  Started    5m19s  kubelet            Started container dnsutils-sa
```

De ahí hay algunas cosas que nos interesan para entender un poco cómo es que ahora nuestro pod accede al token.

Primero, "Mounts". Ahí vemos que hay un volumen del mismo nombre que el secreto generado cuando se crea el Service Account. Además dice estar montado en "/var/run/secrets/kubernetes.io/serviceaccount". 
Más abajo vemos que en "Volumes" efectivamente dice montar el secreto del nombre del ServiceAccount. Veamos dentro del container qué hay. 

```bash
» k exec -it dnsutils-sa -- sh
# cd /var/run/secrets/kubernetes.io/serviceaccount
# ls
ca.crt     namespace  token

#---- Hay algunos archivos ahí. Por el momento nos interesa el 'token'.

# cat token
eyJhbGciOiJSUzI1NiIsImtpZCI6Ing2dGJWczNOSnk5VFA0N2Y5b3cySE56YjM3OGZJcU1wR0YyTWZIbTZQTk0ifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlbW8tc2VydmljZS1hY2NvdW50LXRva2VuLWd3Yjk3Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImRlbW8tc2VydmljZS1hY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiMjM5ZjZiM2ItZjQ0Yi00NGRjLWJjNjAtODlhY2Y1NjAyZTUwIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmRlZmF1bHQ6ZGVtby1zZXJ2aWNlLWFjY291bnQifQ.TLI51AAJBiR2VfwZfIh6NXPHmRdPLvMqXwnnaPZM8xz3S6A65meCm5ZT3uZFIMmDYXvdliRs_2rZb6dAeSXAIKt9uUB9bbG66ZxeTGJzksYKR6sksj-g-MoYZKC5dqOEOXPCpSB13psKV6e2-AGW82sj0HNeKeREh_6vPYSvGQEHmp6E6JqVTk69Xln5AMDJzWIndXHbputuA3Bc9VIilYaA0kUVxGmt4869xHGQduLc22eO50sQfBPK4JD48AZuO9iYVuCMrmUzhgCCeKvAcEzEDXiMCXaBxNz3djSONRcIo4CtzJHhF3m9Exy35hAX0ZfnFZPicVUW91TajP4xNA
```

Ahora entendimos cómo una aplicación puede acceder al token. El comportamiento esperado para el token generado por un service account es ser montado en el path que vimos más arriba. 

Por otro lado, vale la pena aclarar que los secretos, como ya vimos, pueden ser montados en un path deseado, o también pueden ser pasados como variables de entorno en deployments u otros recursos. 

## Auditar el uso de ServiceAccounts
- Comentar herramientas como https://github.com/liggitt/audit2rbac
- Posible kube-polp? 

## Conclusión
### Sobre ServiceAccounts
- Se crea un default Service Account por cada Namespace.
- Varios pods pueden usar el mismo SA. 
- Pods pueden utlizar sólo un SA a la vez. 
- Los permisos del serviceaccount default no dejan listar o modificar recursos. 

### Recomendaciones para el uso de ServiceAccounts
- Dejar el default Service Account tal como está. No agregarle permisos.
- Crear un ServiceAccount por aplicación, otorgando mínimos privilegios. 

# Securizar Componentes
## Control Plane (Master Nodes)
Cuando creamos un clúster de Kubernetes en un cloud provider como por ejemplo AWS por defecto nos va a indicar que los nodos 'master' van a estar públicos. ¿Qué quiere decir esto? Nuestros nodos van a estar en una red pública, con IPs públicas y cualquier persona que conozca la IP/DNS podría hacer requests contra ellos. Por supuesto que, para hacer uso de la ApiServer debemos estar autenticados y aquellos que no tengan credenciales/token no podrán hacer uso de ninguna función. 

Ya vimos como podemos obtener un token desde adentro de un container. Pero veamos el siguiente ejemplo.

Tenemos una aplicación desplegada en nuestro cluster. Esta aplicación tiene una vulnerabilidad y permite al atacante leer archivos del filesystem. Si puede leer el token, luego puede usarlo para hacer pedidos contra la ApiServer desde donde sea ya que es pública en internet. Tendrá los mismos permisos que tenga nuestra aplicación. 

¿Como podemos evitar esto?

Tenemos varias cosas para hacer. Lo primero hacer que nuestros nodos del Control Plane no sean públicos. De este modo sólo desde determinadas IPs que estén 'allowed' en algún firewall o conectados a través de una VPN algún usuario va a poder hacer uso de la ApiServer. 

De este modo nuestro atacante, por más de que tenga el token no va a poder acceder a la ApiServer ya que está fuera de su alcance. 

### ETCD
ETCD es la base de datos donde se guardan las configuraciones y eventos de nuestro clúster de Kubernetes. ETCD no tiene por qué vivir directamente dentro de los master nodes. Si bien la mayoría de las distribuciones de Kubernetes provistas por cloud providers nos abstraen de la seguridad de ETCD, en caso de implementar nuestra propia instalación de Kubernetes debemos seguir los mismos pasos que tomamos con los master nodes.

- Dejar ETCD detrás de un firewall donde sólo IPs conocidas puedan accedeer (probablemente sólo las de los master nodes).
- Habilitar autenticación en ETCD. 
- Habilitar encriptación de la data en la base de datos. Si alguien llega, que se encuentre con data que no puede leer.

## Hosts
Así como mencionamos posibles vulnerabilidades en nuestras aplicaciones que pueden permitir a potenciales atacantes leer tokens dentro del FileSystem, podemos encontrarnos con más casos. 

Una de las posibilidades es que puedan escalar desde el container al host/nodo/worker por algún bug en el kernel o de kubernetes y puedan hacer cosas como:
- Usar otros servicios que viven en otros pods del cluster (Fix con NetworkPolicies, más adelante)
- Crear nuevos containers con aplicaciones dentro del host o correr procesos. O leer información sensible de containers que están actualmente corriendo, como secretos, contraseñas o configuraciones. 
- Leer token del Kubelet (Kubernetes Agent instalado en cada nodo que permite registrar el nodo en la ApiServer) y potencialmente impersonarlo. De este modo podría conseguir más información sobre el clúster. 

¿Qué medidas podemos tomar para evitar esto?
- Correr las aplicaciones en los containers como non-root.

```yaml
spec:
    securityContext:
        runAsUser: 1000
```
- Read Only FileSystem
De esta manera evitamos que archivos que se escriban archivos de sistema que no esperamos que sean modificados. 
(Por ejemplo el reemplazo de algún binario que nuestra app utiliza por otro que ejecuta código malicioso.)

```yaml
spec:
    securityContext:
        readOnlyRootFilesystem: true
```

- Not allowing privilege escalation.
```yaml
spec: 
    securityContext:
        allowPrivilegeEscalation: false
```

¿Como forzar estas reglas para todos los pods que deployemos?
Puede pasar que varios equipos o usuarios que usen el cluster para desplegar sus aplicaciones sigan las recomendaciones que mecionamos recién a través de 'AdmissionControllers'.

##_imagen_flow_validation_mutation_

## Tráfico entre Pods
En uno de los temas anteriores, en el que un atacante lograba ejecutar código desde uno de nuestros containers o lograba acceso al host/nodo/worker, dijimos que era posible que tenga acceso a usar otros servicios que corren en el cluster, usualmente servicios HTTP.

Para ver cómo podemos solucionar este problema vamos a meternos un poco más a fondo en cómo funciona el networking en Kubernetes.

## Networking
Kubernetes permite la comunicación entre sus componentes _containerizados_. Existen varias implementaciones del modelo de Networking para Kubernetes, las más conocidas son Flannel, Cillium o Calico. 

Los cloud providers como AWS no ofrecen por defecto el _feature_ de 'NetworkPolicies' que mencionamos antes. Por lo que no tenemos una manera obvia de negar tráfico entre servicios dentro del clúster.  

Como Kubernetes tiene varias abstracciones para orquestar containers (pods, services, ingresses) cada una de ellas implica una manera de comunicarse con otra. 


## Container a Container dentro de un Pod.
Los containers en Kubernetes están agrupados en pods. Un pod puede contener más de un sólo container. La solución de Networking que hayamos elegido va a asignar una IP única en todo el clúster a ese pod. Los containers dentro de ese pod pueden comunicarse entre sí utilizando `localhost`. Dado que un Pod posee una única IP, todos los containers del mismo pod deben vivir en el mismo nodo del cluster. 

Para eso vamos a deployar el siguiente Pod en un clúster (en este caso por cuestiones de implementación de networking, minikube no nos va a funcionar.) 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:
  containers:
  - name: hello-world
    image: vad1mo/hello-world-rest
    ports:
     - name: http
       containerPort: 5050
  - name: dns-utils
    image: gcr.io/kubernetes-e2e-test-images/dnsutils:1.3
    command:
      - sleep
      - "3600"
```

Este pod contiene dos containers. Uno con DNS Utils que ya lo conocimos antes, y otro con una API de ejemplo, expuesta en el puerto 5050.

Nos metemos en el container de DNS Utils a ver qué IP tenemos:

```bash
 » k exec -it pod/two-containers --container dns-utils -- sh
/ # ip a | grep /32
    inet 10.64.232.152/32 scope global eth0
```

Container de 'Hello-world':
```bash
» k exec -it pod/two-containers --container hello-world -- sh
/app # ip a | grep /32
    inet 10.64.232.152/32 scope global eth0
```

Parece que la [documentación]https://kubernetes.io/docs/concepts/workloads/pods/#pod-networking) no nos mintió.

## Pod a Pod
Vamos a entrar en nuestro otro pod de dnsutils existente en el cluster, no en el que creamos en el pod 'two-containers', y vamos a hacer un pedido a otro pod, en este caso al de 'hello-world' que trabajamos antes.

```bash
» k exec -it pod/dnsutils -- sh
/ # ip a | grep /32
    inet 10.64.232.168/32 scope global eth0

/ # curl 10.64.232.152:5050
/ - Hello World! Host:two-containers/10.64.232.152/
```

Pudimos llegar perfectamente desde un pod al otro. Nada nos impidió el acceso. 
Por supuesto que la idea no es utilizar IPs y puertos ya que conocemos de hostnames. Además, en Kubernetes podemos tener múltiples réplicas de un mismo pod, es decir 10 Pods corriendo la misma aplicación, por lo que no quisiera apuntar a uno solo de ellos. Dada la naturaleza dinámica de un entorno corriendo Kubernetes podemos decir que esas IPs que tienen asignadas los Pods no van a estar mucho tiempo relacionadas. Para resolver estos problemas existe el Service.

## Service to Pod
El objeto 'Service' es un objeto de Kubernetes que nos deja manejar el componente 'Kube-proxy' permitiéndonos crear proxys reversos dentro de nuestro cluster y habilitando la posibilidad de que los pods 'detrás' de este Service puedan ser intercambiados o actualizados con dinamicamente. 

Esta vez vamos a crear dos objetos. Un Deployment (conjunto de Pods) y un Service. 
```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: hello-world-with-svc 
  namespace: default
  labels:
    app: hello-world
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world 
        type: default
    spec:
      containers:
      - image: vad1mo/hello-world-rest
        name: hello-world
        ports:
        - name: http
          containerPort: 5050
---
kind: Service
apiVersion: v1
metadata:
  name: hello-world-svc
spec:
  selector:
    app: hello-world
  ports:
  - port: 80
    targetPort: 5050
  
```

Veamos el Service que creamos.

```bash
» k get services
NAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
hello-world-svc   ClusterIP   172.20.190.244   <none>        80/TCP    7m42s
```

Inspeccionemos un poco más.

```bash
» k describe service hello-world-svc
Name:              hello-world-svc
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          app=hello-world
Type:              ClusterIP
IP Families:       <none>
IP:                172.20.190.244
IPs:               <none>
Port:              <unset>  80/TCP
TargetPort:        5050/TCP
Endpoints:         10.64.232.162:5050
Session Affinity:  None
Events:            <none>
```

Por lo que vemos asignó una IP a nuestro Service y en el puerto 80 escucha. Podemos probar desde otro pod a ese Service a ver si llegamos.

```bash
» k exec -it pod/dnsutils -- sh
/ # curl 172.20.190.244
/ - Hello World! Host:hello-world-with-svc-7cd6d8dfc6-t4g9p/10.64.232.162/ #
```

Llegamos bien a través de nuestro Service. Además, podemos llegar a él a través de un DNS generado por Kubernetes.

```bash
# curl hello-world-svc.default.svc.cluster.local
/ - Hello World! Host:hello-world-with-svc-7cd6d8dfc6-t4g9p/10.64.232.162/ # 
```

- Prueba a través de distintos namespaces. 
##TODO

El tráfico está bastante libre entre los pods. Cualquiera puede llegar a cualquier lado. Vamos a ver cómo podemos restringirlo.

## Network Policies
Probamos network policies usando Calico. Lo necesario para instalarlo está en `./networking/calico-install`. 

Vamos a probar la demo de Calico que a través de una UI nos permite ver cómo se comunican los servicios. 

 - En el namespace 'star' tenemos el frontend y el backend de la aplicación.
 - En el namespace 'client' tenemos al cliente que va a querer usar la aplicación.
 - En el namespace 'management-ui' tenemos la aplicación que vamos a utilizar para monitorear la accesibilidad entre los pods/servicios.

Para empezar a ver esto en acción exponemos el service del 'management-ui' a nuestro localhost y vamos al navegador a ver qué tenemos.

```bash
» k port-forward service/management-ui -n management-ui 9001
```

Si ingresamos en http://localhost:9001 deberíamos ver lo siguiente:

![image](./networking/network-policies-demo/images/stars-default.png)

En el gráfico vemos B por Backend, F por Frontend y C por Client. Las líneas que conectan el grafo nos indican que todos los nodos se pueden ver entre sí. 

Vamos a aplicar unas NetworkPolicies para lograr que las aplicaciones no se vean entre sí.  Pero antes vamos a revisarlas. `./networking/network-policies-demo/isolate-services/`   

En client-deny.yaml tenemos lo siguiente: 

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: client
  name: default-deny
spec:
  podSelector:
    matchLabels: {}
```

Una NetworkPolicy vacía, por lo que nada va a poder llegar al Namespace `client`. 

En stars-deny.yaml:

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: default-deny
  namespace: stars
spec:
  podSelector:
    matchLabels: {}
```
Otra NetworkPolicy vacía. 

Aplicamos.

```bash
 » k apply -f isolate-services/ 
```

Y ahora al refrescar la pestaña del navegador deberíamos no ver nada, ya que perdimos acceso desde el namespace al que estamos accediendo 'management-ui' al resto al aplicar las políticas. 

Como nosotros queremos ver qué está pasando entre los componentes, vamos aplicar una NetworkPolicy que nos permita volver a acceder a la UI como hicimos antes. `./networking/network-policies-demo/allow-ui-to-services`.

Antes de aplicar veamos que tenemos. 

allow-ui.yaml
```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: stars
  name: allow-ui
spec:
  podSelector:
    matchLabels: {}
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              role: management-ui
```
Con esta NetworkPolicy estamos permitiendo el ingreso a los recursos en el namespace 'stars' (frontend y backend) desde aquellos que estén en el namespace 'management-ui'. 

Con este permiso ya podríamos ver desde la UI a dos de los servicios. Además ellos se ven entre sí.

```bash
» k apply -f allow-ui.yaml   
```

allow-ui-client.yaml
```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: client 
  name: allow-ui 
spec:
  podSelector:
    matchLabels: {}
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              role: management-ui 
```
Con esta NetworkPolicy estamos permitiendo el ingreso a los recursos en el namespace 'client' (client) desde aquellos que estén en el namespace 'management-ui'.

Aplicamos.

```bash
» k apply -f allow-ui-client.yaml 
```
Ahora podríamos también ver al cliente desde nuestra UI.

![image](./networking/network-policies-demo/images/stars-no-traffic.png)

Como último paso vamos a dejar que nuestro Cliente vea al frontend, pero no al backend. Y que el frontend sí vea al backend. 

Revisamos los archivos que están en `./networking/network-policies-demo/last-step/`

backend-policy.yaml

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: stars
  name: backend-policy
spec:
  podSelector:
    matchLabels:
      role: backend
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: frontend
      ports:
        - protocol: TCP
          port: 6379
```

En esta NetworkPolicy habilitamos el tráfico desde los pods que tengan como label 'frontend' a los que tengan el label 'backend'. Adicionalmente, aclaramos que el tráfico que habilitamos es sólo al puerto 6379.

frontend-policy.yaml

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: stars
  name: frontend-policy
spec:
  podSelector:
    matchLabels:
      role: frontend 
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              role: client
      ports:
        - protocol: TCP
          port: 80
```

Para la NetworkPolicy del frontend, en el namespace de 'stars', habilitamos el tráfico desde el namespace 'client'. Y en este caso permitimos el tráfico sólo al puerto 80, que es el puerto donde sirve nuestra aplicación de Frontend. 

Aplicamos todo esto.

```bash
» k apply -f last-step/
```

Como resultado de esto tendríamos que ver que el cliente se ve con el frontend, pero no con el backend, y que sólo el frontend (y nuestra UI) pueden llegar al backend. 

![image](./networking/network-policies-demo/images/stars-final.png)

En resumen, las NetworkPolicies:
- Nos dejan controlar tráfico entre Pods.
- Nos dejan controlar tráfico entre Pods según su namespace.
- Nos dejan controlar tráfico entre Pods/Services según su puerto.
- Por defecto, una Policy vacía es un DenyAll. 
- Por el momento, sólo podemos agregar AllowRules. 
- No nos dejan auditar conecciones aceptadas o negadas.

Más info sobre NetworkPolicies: https://kubernetes.io/docs/concepts/services-networking/network-policies/

_Luego de terminar la demo, borrar todo_
```bash
» k delete -f install 
```

# Auditoría 
_Para este apartado necesitamos activar 'Audit' en nuestro cluster, para eso tenemos que reiniciar minikube_

```
minikube stop

mkdir -p ~/.minikube/files/etc/ssl/certs

cat <<EOF > ~/.minikube/files/etc/ssl/certs/audit-policy.yaml
# Log all requests at the Metadata level.
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
EOF

minikube start \
  --extra-config=apiserver.audit-policy-file=/etc/ssl/certs/audit-policy.yaml \
  --extra-config=apiserver.audit-log-path=-
```
_Más info en: https://minikube.sigs.k8s.io/docs/tutorials/audit-policy/_

Necesitamos logs!
Podemos distinguir dos tipos de logs dentro de Kubernetes. Por un lado los logs generados por las aplicaciones desplegadas dentro del clúster, que típicamente envían logs por stdout y soluciones externas _scrappean_ esos logs y los llevan a otro lado para almacenarlos y eventualmente analizarlos.

Nosotros nos vamos a enfocar no en logs de aplicaciones sino en logs de Kubernetes en sí mismo. 

El caso que querríamos resolver es el siguiente: Tenemos un clúster, dentro de ese clúster hay un secreto en un namespace que tiene un valor determinado. De un momento para el otro el valor de ese secreto cambió. Eso nos rompe el funcionamiento de algunas aplicaciones, alguien en nuestro equipo se da cuenta, lo arregla y todo vuelve a la normalidad. Nosotros nos preguntamos... ¿Qué pasó con ese secreto?, ¿Quién cambió el valor del secreto?.

Si empezamos a ver logs de los pods asociados a ese secreto lo único que vemos son errores, a lo sumo podemos determinar la hora a la que todo empezó a fallar, pero como no es responsabilidad de la aplicación desplegar el secreto, ella no sabe quién pudo haberlo cambiado. Los logs de aplicaciones no nos sirven para este caso.

También podemos ejecutar 'events'. Si ejecutamos `kubectl get events` y buscamos un poco vamos a dar con información acerca de que el secreto fue modíficado. 

Quien sí tiene la información que buscamos es nuestra ApiServer.
https://www.youtube.com/watch?v=HXtLTxo30SY

Filtramos un poco los logs del ApiServer: 

```bash
» kubectl logs kube-apiserver-minikube -n kube-system | grep pochoclo | jq .  
{                                                                    
  "kind": "Event",    
  "apiVersion": "audit.k8s.io/v1",
  "level": "Metadata",
  "auditID": "c281c0da-f1c4-44ba-9419-91fb53041e82",
  "stage": "ResponseComplete",
  "requestURI": "/api/v1/namespaces/kube-system/pods/kube-apiserver-minikube",
  "verb": "get",
  "user": {
    "username": "pochoclo",                                          
    "groups": [                                                      
      "group1",   
      "system:authenticated"                                         
    ]                                                                                                                                      
  },
  "sourceIPs": [
    "192.168.49.1"                                                                                                                         
  ],
```

- Como mandar los logs a otro lado - Mención a feature alpha.
- Mención a como activar audit logs
- Mención especial a ebpf.


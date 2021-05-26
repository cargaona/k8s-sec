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
Dentro de las reglas, tenemos 'apiGroups': Dentro de la ClusterAPI a qué grupos de endpoints podemos hacer pedidos. Más info sobre los API Groups existentes [acá](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.21/#-strong-api-groups-strong-). En este caso al dejarlas como "" estamos dando acceso a todos los API Groups.

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


## Cómo funciona el serviceAccount? Kubernetes ClusterAPI.

Como vimos al principio, la manera de interactuar con Kubernetes es a través de su API y las aplicaciones no son la excepción. Podemos intentar hacer un request a la Kubernetes API para ver qué nos devuelve. 

Para esta prueba vamos a utilizar una imagen de docker conocida como 'dns-utils' que nos permite troubleshootear problemas de DNS y Networking dentro de un cluster. Podés instalarla del siguiente modo:
```bash
kubectl apply -f https://k8s.io/examples/admin/dns/dnsutils.yaml
```

Una vez _deployado_ el dns-utils podemos ejecutar el siguiente comando para ver qué nos devuelve la Kubernetes API.

_Si te genera dudas cómo es que sólo "kubernetes" resuelve a la Cluster API, podés correr el siguiente comando y entender a dónde está resolviendo:_
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

Parece que llegamos a la ClusterAPI sin problemas pero no tenemos permisos, por eso obtuvimos un 403. Ahora es donde entra en juego el #service-account-token.

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

## ClusterAPI request 2. 

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

Parece que esta vez tuvimos éxito. Ya vimos como crear un ServiceAccount y usar el token generado para utilizar la Kubernetes ClusterAPI. Pero hasta ahora estuvimos tomando ese token a mano y ejecutando requests por nosotros mismos. ¿Cómo obtienen estos permisos las aplicaciones? 


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

# Networking
Kubernetes permite la comunicación entre sus componentes _containerizados_. Existen varias implementaciones del modelo de Networking para Kubernetes, las más conocidas son Flannel o Calico. 

Como Kubernetes ofrece varias abstracciones para orquestar containers (pods, services, ingresses) cada una de ellas implica una manera de comunicarse con otra. 

## Container a Container
Los containers en Kubernetes están agrupados en pods. Un pod puede contener más de un sólo container. La solución de Networking que hayamos elegido va a asignar una IP única en todo el clúster a ese pod. Los containers dentro de ese pod pueden comunicarse entre sí utilizando `localhost` 


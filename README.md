# WebLogic sur OpenShift (mode Domain-in-PV)
## Introduction
Ce document présente un exemple du [WebLogic Kubernetes Operator](https://github.com/oracle/weblogic-kubernetes-operator) sur OpenShift utilisant le mode [domain in pv](https://oracle.github.io/weblogic-kubernetes-operator/userguide/managing-domains/choosing-a-model/). Ce guide est basé sur le document suivant: [ WebLogic Operator documentation](https://oracle.github.io/weblogic-kubernetes-operator/quickstart/) et [ce blog de Mark Nelson](https://blogs.oracle.com/weblogicserver/running-weblogic-on-openshift). 

Ce document présente une variante d'utilisation de l'Operator WebLogic.  Pour une autre variante, consultez le répertoire suivant pour le [domain in image](https://github.com/jnovotni/weblogic-operator-on-openshift)

## Pré-requis
- Vous devez avoir un compte autorisé à accéder au registre de conteneur d'Oracle. Cela vous permet d'aller chercher l'image de base utilisée dans les étapes ci-dessous.

Si vous n'avez pas encore de compte, vous pouvez vous rendre sur https://container-registry.oracle.com et en créer un.

Vous devez également accepter les conditions d'utilisation de l'image WebLogic à partir du Registre d'Oracle.

- Vous devez avoir les outils de ligne de commande OpenShift installés (incluant `oc` et `kubectl`). 
Vous pouvez télécharger la dernière version des outils à partir de
 [ce lien](https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable-4.7/) ou télécharger à partir des menus de votre installation OpenShift.

- Vous devez avoir Helm (v3) installé sur votre machine locale. Vous pouvez trouver des instructions pour installer l'outil ici:
 [ce lien](https://helm.sh/docs/helm/helm_install/) ou télécharger à partir des menus de votre installation OpenShift.

- Vous devez disposer d'un accès `cluster-admin` à votre cluster OpenShift.

### Partie 1: Cloner ce répertoire GitHub

Sur votre ordinateur local, clonez ce dépôt en exécutant la commande suivante :

```
git clone --recurse-submodules https://github.com/masauve/weblogic-operator-on-openshift.git
```

l'option --recurse-submodules est nécessaire. Le dépôt d'Oracle pour l'operator WebLogic est cloné dans un sous-répertoire et utilisé par cette procédure d'installation.


Ensuite, changez de répertoire vers le référentiel nouvellement cloné en exécutant la commande suivante :

```
cd weblogic-operator-on-openshift/
```

### Partie 2: Déployer l'Operator WebLogic 

Authentifiez vous sur OpenShift en utilisant un compte cluster-admin

La commande sera similaire à :

```
oc login --token=sha256~xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx --server=https://api.ocp.example.com:6443
```

Créez un projet (namespace) dans lequel installer l'opérateur WebLogic Kubernetes en exécutant la commande suivante :

```
oc new-project weblogic-operator
```

Créez un compte de service que l'opérateur pourra utiliser en exécutant la commande suivante :

```
oc create serviceaccount -n weblogic-operator weblogic-operator-sa
```

Installez l'opérateur à l'aide de Helm en exécutant la commande suivante :

```
helm install weblogic-operator weblogic-kubernetes-operator/kubernetes/charts/weblogic-operator \
  --namespace weblogic-operator \
  --set image=ghcr.io/oracle/weblogic-kubernetes-operator:3.2.2 \
  --set serviceAccount=weblogic-operator-sa \
  --set "enableClusterRoleBinding=true" \
  --set "domainNamespaceSelectionStrategy=LabelSelector" \
  --set "domainNamespaceLabelSelector=weblogic-operator\=enabled" \
  --wait
```

En cas de succès, la réponse sera similaire à :

```
NAME: weblogic-operator
LAST DEPLOYED: Wed May 19 11:30:49 2021
NAMESPACE: weblogic-operator
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

La commande Helm que nous avons exécutée a configuré l'opérateur pour gérer les domaines dans n'importe quel projet OpenShift (espace de noms) avec l'étiquette « weblogic-operator=enabled ».

Validez le déploiement de l'opérateur en exécutant la commande suivante :

```
oc get pods -n weblogic-operator
```

Le résultat devrait ressemblé à ceci:

```
NAME                                 READY   STATUS    RESTARTS   AGE
weblogic-operator-6bb58697b7-2zbql   1/1     Running   0          87s
```

Assurez-vous que le pod est « Running » et « READY (1/1) ».


### Partie 3: Création d'un projet pour le domaine WebLogic

Créez un projet (namespace) sur lequel déployer le domaine WebLogic en exécutant la commande suivante :

```
oc new-project sample-domain1
```

Étiquetez le projet (namespace) pour vous assurer que l'opérateur puisse le gérer.

```
oc label ns sample-domain1 weblogic-operator=enabled
```

L'opérateur s'attend à ce qu'un secret Kubernetes existe avec les informations d'identification de l'administrateur WebLogic. Le mot de passe **doit** comporter au moins 8 caractères alphanumériques avec au moins un chiffre ou un caractère spécial. Si vous ne respectez pas cette exigence, la création du domaine échouera.

Pour créer un secret à l'aide des identifiants par défaut, exécutez la commande suivante :

```
./weblogic-kubernetes-operator/kubernetes/samples/scripts/create-weblogic-domain-credentials/create-weblogic-credentials.sh \
-u administrator -p AbCdEfG123! -n sample-domain1 -d domain1
```

Le résultat sera similaire à :

```
secret/domain1-weblogic-credentials created
secret/domain1-weblogic-credentials labeled
The secret domain1-weblogic-credentials has been successfully created in the sample-domain1 namespace.
```


### Partie 4 "DOMAIN IN PV"

Créer un répertoire à partir du répertoire courant et copier le fichier de configuration 'exemple' du domaine:

```
mkdir domain-in-pv
cp weblogic-kubernetes-operator/kubernetes/samples/scripts/create-weblogic-domain/domain-home-on-pv/create-domain-inputs.yaml domain-in-pv/
```

Modifier les paramêtres au besoin (nom du domaine, nombre d'instances...)

La paramêtre 'namespace' doit être modifié:

```
### remplacer:

namespace: default

### par:

namespace: sample-domain1 
```

Sous OpenShift, le fichier suivant doit être modifié:

```
weblogic-kubernetes-operator/kubernetes/samples/scripts/create-weblogic-domain/domain-home-on-pv/create-domain-job-template.yaml
```

Supprimer la section suivante:

```
      initContainers:
        - name: fix-pvc-owner
          image: %WEBLOGIC_IMAGE%
          command: ["sh", "-c", "chown -R 1000:0 %DOMAIN_ROOT_DIR%"]
          volumeMounts:
          - name: weblogic-sample-domain-storage-volume
            mountPath: %DOMAIN_ROOT_DIR%
          securityContext:
            runAsUser: 0
            runAsGroup: 0
```

Créer un PVC à partir de la console OpenShift:
```
Nom:  domain1-weblogic-sample-pvc
Type: RWX
```
Préparer pour la création du domaine:

```
./weblogic-kubernetes-operator/kubernetes/samples/scripts/create-weblogic-domain/domain-home-in-pv/create-domain.sh -i domain-on-pv/create-domain-inputs.yaml -o domain-in-pv/
```

Créer le domaine et créer les routes: 
```
oc apply -f kubectl /domain-in-pv/weblogic-domains/domain1/domain.yaml

oc expose svc/domain1-cluster-cluster-1
oc expose svc/domain1-admin-server

```

Vous pouvez maintenant déployer des applications et configure WebLogic avec la console web ou WLST.

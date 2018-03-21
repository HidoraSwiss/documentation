# Script de déploiement sur Hidora

Grâce à ce script de déploiement, il est possible de gérer plus finement les déploiement sur Hidora (ou autre hoster Jelastic).

Placé avec les sources d'un projet dans un dépôt Git, il peut être exécuté dans un pipeline automatisé pour faire du *Continuous Deployement*.

## Script

```bash
#!/bin/bash
#
# Deploy script for Hidora / Jelastic
# Use: ./deploy-to-jelastic.sh <user hidora> <pass hidora> <env-name>
#

# Configuration
HOSTER_URL="https://app.hidora.com" # Jelastic hoster to use
ENV_NAME=${3:-$ENVNAME} # Environment name for deploy. If exists, it must be owned by the given user
MANIFEST_PATH=${4:-manifest.jps} # Use manifest.jps in the current dir for environment creation
DEPLOY_GROUP=${5:-cp} # By default, redeploy nodes from the "cp" group in environment

# Constants
APPID="1dd8d191d38fff45e62564fcf67fdcd6"
CONTENT_TYPE="Content-Type: application/x-www-form-urlencoded; charset=UTF-8;";
USER_AGENT="Mozilla/4.73 [en] (X11; U; Linux 2.2.15 i686)"

echo "SignIn...";
signIn=$(curl -H "${CONTENT_TYPE}" -A "${USER_AGENT}"  -X POST \
-fsS "${HOSTER_URL}/1.0/users/authentication/rest/signin" -d "login=$1&password=$2");
echo 'Response signIn user: '$signIn
RESULT=$(jq '.result' <<< $signIn );
SESSION=$( jq '.session' <<< $signIn |  sed 's/\"//g' );

echo "Check if env is created...";
envs=$(curl \
-H "${CONTENT_TYPE}" \
-A "${USER_AGENT}" \
-X POST \
-fsS ${HOSTER_URL}/1.0/environment/control/rest/getenvs -d "appid=${APPID}&session=${SESSION}");
CREATED=$(echo $envs | jq '[.infos[].env.envName]' | jq "contains([\"$ENV_NAME\"])")

# If environment exists, it is redeployed
if [[ "${CREATED}" = "true" ]]; then
  echo "Redeploy existing environment (Group ${DEPLOY_GROUP})..."
  redeploy=$(curl \
  -H "${CONTENT_TYPE}" \
  -A "${USER_AGENT}" \
  -X POST \
  -fsS ${HOSTER_URL}/1.0/environment/control/rest/redeploycontainersbygroup \
  -d "appid=${APPID}&session=${SESSION}&envName=${ENV_NAME}&tag=latest&nodeGroup=${DEPLOY_GROUP}&useExistingVolumes=true&delay=20");
  echo "Environment redeployed"
# If environment doesn't exist, it is created using the given manifest
else
  echo "Install new environment...";
  MANIFEST=$(cat $MANIFEST_PATH)
  installApp=$(curl \
  -A "${USER_AGENT}" \
  -H "${CONTENT_TYPE}" \
  -X POST -fsS ${HOSTER_URL}"/1.0/development/scripting/rest/eval" \
  --data "session=${SESSION}&shortdomain=${ENV_NAME}&envName=${ENV_NAME}&script=InstallApp&appid=appstore&type=install&charset=UTF-8" --data-urlencode "manifest=$MANIFEST");
  echo "Response install env: "$installApp;
fi

exit 0
```

## Requirements

Le script utilise le programme `jq` pour traiter les données renvoyées par l'API de Jelastic. Il est donc nécessaire de l'installer au préalable: https://stedolan.github.io/jq/

Il est également nécessaire d'avoir `curl` installé. C'est déjà le cas dans la plupart des systèmes mais cela doit peut être se faire manuellement dans certains cas, notamment avec des images Docker Alpine.

## Utilisation

Dans une utilisation simple, il suffit de fournir ses identifiants Hidora et le nom de l'environnement de déploiement:

```bash
./deploy-to-hidora.sh <email> <password> <env-name> 
```

**Exemple:** `./deploy-to-hidora.sh test@hidora.com mypass demonstration`

> If the env name is already taken, you need to be the owner of the environment in ordrer to redeploy.

Par défaut, si l'environnement n'existe pas, le script va utiliser le fichier `manifest.jps` dans le dossier courant pour en créer un nouveau.
Si l'environnement existe, il va re-deployer les conteneurs principaux (groupe *cp*) en allant chercher à nouveau l'image à sa source (Docker Hub ou Docker registry privé).

Dans une utilisation avancée, il est possible de préciser quel fichier de manifest utiliser et quel groupe de conteneurs re-deployer.

```bash
./deploy-to-hidora.sh <email> <password> <env-name> <manifest-file> <group-name>
```

**Exemple:** `./deploy-to-hidora.sh test@hidora.com mypass demonstration my-infra.json bl`

> Plus d'infos sur les grouppes de conteneurs ici : https://docs.cloudscripting.com/creating-manifest/selecting-containers/


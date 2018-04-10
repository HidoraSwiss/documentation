# Deployment script on Hidora

Thanks to this script, it is possibe to manage more finely deployments on Hidora (or other Jelastic hoster).

Disposed with the code source of your project in a Git repository, it can be executed in an automated pipeline to make *Continuous Deployement*.

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

The script uses the program `jq` to treat data returned by the API of Jelastic. So, you need to install it before:
https://stedolan.github.io/jq/

It is also necessary to have `curl` installed. It is already present on the most systems but it is possible that you will have to install it manually, more especially with Docker Alpine images.

## Usage

In a simple use, you just need to specify your Hidora credentials and the name of your future environment.

```bash
./deploy-to-hidora.sh <email> <password> <env-name>
```

**Example:** `./deploy-to-hidora.sh test@hidora.com mypass demonstration`

> If the env name is already taken, you need to be the owner of the environment in ordrer to redeploy it.

By default, if the environmnent doesn't exist already, the script will use the file `manifest.jps` in the current directory to create a new one.
If the environment exists, it will redeploy the main container (*cp* group) from the source image (Docker Hub or Private Docker registry).

In an advanced use, it is possible to specify the manifest file to take and the node group (cp, lb, db ...) of the container to redeploy it.

```bash
./deploy-to-hidora.sh <email> <password> <env-name> <manifest-file> <group-name>
```

**Example:** `./deploy-to-hidora.sh test@hidora.com mypass demonstration my-infra.json bl`


>More information about node groups here : https://docs.cloudscripting.com/creating-manifest/selecting-containers/


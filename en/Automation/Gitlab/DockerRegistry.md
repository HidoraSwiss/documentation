# Deployment with Docker Registry

![Déployment with docker registry](../../../images/Registry Pipeline.png)

This method is more advanced than [git+ssh method](/Automation/Gitlab/ssh+git), use Docker images and Docker Registry for test and deployment. It is more adapted for complex environment thanks toits benefits, in particulary : 

- We use fully fully the functionality of the Docker containers. In this way, sources and dependencies of your application will be always together.
- It is possible to test our application in a pipeline because we can create quickly a test environment.
- It is easy to debug a potentiel issue because we have the possibility to collect on a localhost an image located in a Docker registry.

> From a developer point of view, it is still easy to launch an automate pipeline via a simple git push.

## Prerequisite

- A Gitlab Server with a runner configured as *Docker* mode. More information on this [documentation](https://docs.gitlab.com/runner/install/docker.html).
- A Docker Registry Server (hosted on Hidora). You can use this script, to deploy it in few sconds on our platform:
https://github.com/HidoraSwiss/manifest-registry

## Dockerfile

Firstly, it is necessary to add on your files sources `Dockerfile`  which will build an image, ready for Production for every push.

We add files sources of the Git repository directly on our image via Dockerfile.
For example, if you want to deploy a PHP application, your Dockerfile will looks like this : 
```dockerfile
FROM php:7.0-apache
COPY . /var/www/html/
```
Note that the usage of `COPY`, will copy all files sources in the directory `/var/www/html` of the Apache Server. For applications more complex (donwload configs, assets compilation ...), you can provide actions to achieve in the Docker file. For more information, have a look on [Dockerfile documentation](https://docs.docker.com/engine/reference/builder/).

## Create the environment

Before to deploy automatically your modification, you need to create an environmnet on Hidora.

For that, start to build an image with your source code locally in order to push it on your Docker Registry:
```bash
docker build -t <url-registry>:<port-registry>/dir/image_name .
docker push <url-registry>:<port-registry>/dir/image_name
```

Now, you have a first image which will allowed you to create an environment on Hidora.

Then, sign into Hidora's web interface and use the oanel *New environment* to create an environment using your image in your Docker Registry.

![Create a container from an image in a Docker Registry](../../../images/Screenshot - image from registry.png)

## .gitlab-ci.yml

In a file `.gitlab-ci.yml`  located in your source code, copy the content below:

```yaml
image: docker

variables:
  IMAGE_TEST: dir/image_name:$CI_COMMIT_REF_NAME
  IMAGE_RELEASE: dir/image_name:latest

stages:
  - build
  - test
  - release
  - deploy

before_script:
  # We need to be authenticate to our Docker Registry during each steps
  - docker login -u $REGISTRY_USER -p$REGISTRY_PASS $REGISTRY_URL

build:
  stage: build
  script:
        # We build the test image
    - docker build -t $REGISTRY_URL/$IMAGE_TEST .
    # We push this image into the Docker registry
    - docker push $REGISTRY_URL/$IMAGE_TEST

test:
  stage: test
  script:
        # We test the latest version of our image application test
        - echo "Test your application using the new image"
        # # Example:
    # - docker pull $REGISTRY_URL/$IMAGE_TEST
    # - docker $REGISTRY_URL/$IMAGE_TEST ./test-script.sh

release:
  stage: release
  only:
    - master
  script:
  # If the image passed the test, we update the image on the Registry
    - docker pull $REGISTRY_URL/$IMAGE_TEST
    - docker tag $REGISTRY_URL/$IMAGE_TEST $REGISTRY_URL/$IMAGE_RELEASE
    - docker push $REGISTRY_URL/$IMAGE_RELEASE

deploy:
  stage: deploy
  image: mwienk/jelastic-cli
  when: manual # The deployment need to be launch manually
  only:
    - master
  before_script:
        # We need to be identified to use Jelastic API on Hidora
    - /root/jelastic/users/authentication/signin --login $LOGIN --password $PASSWORD --platformUrl app.hidora.com
  script:
        # We launch the redeployment of our container on Hidora
    - /root/jelastic/environment/control/redeploycontainerbyid --envName $ENVNAME --nodeId $NODE_ID --tag latest --useExistingVolumes false
```
This configuration describe a general pipeline that need to be adapted to your application. Think to modify : 

- The name of your image in variables `dir/image_name`
- Commands line to launch during `test` spteps

Then, you need to provide to your pipeline different environment variables to be able to deploy corectly your application. From your repository Gitlab, go into *Settings > CI/CD > Secret variables* and add these next variables: 

- **REGISTRY_URL**: Adress of your Docker Registry. For example, `registry.hidora.com`

- **REGISTRY_USER** et **REGISTRY_PASS**: Credentials of your Docker Registry.
- **ENVNAME**: Environment name that you want to use to deploy your application. For example, `env-542623` (do not put the  *.hidora.com* part)
- **NODE_ID**: Node ID, where you want to deploy your app.
- **LOGIN** et **PASSWORD**: Credentils used to sign in on Hidoras's plateform.

> To manage finely a deployment on Hidora, a documentation are available [here](/Automatisation/Script de déploiement).


## Let's Go !

Once all steps have been realised, push your source code on Gitlab with your new files and check the tab *CI/CD > Pipelines* of your Gitlab repository to see logs.

![State of a Pipeline completed](../../../images/Screenshot - deploy line.png)

If everything works well, you will see that each "job" of your pipeline show a green icon. Depending the configuration of your Gitlab, the last job is on "manual" mode, it means that you need to launch it manually for the deployment. To complete the pipeline, click on the "play" symbol.

In contrast, click on the job which is not completed to see the cause.

## Data persistence

Depending the application, there is few configurations that can be implemented to avoid to loose data.

In the deployment method above, if files are added on the Deployment Server, it will be lost during the redeployment. It is because you use the option `--useExistingVolumes false` (last line of the pipeline) which  will replace all files of the containers by the new image.

To have persistence data, for example files which have been uploaded or logs files, you need to configure your Dockerfile like below: 

-In your Dockerfile, repository that you do not want to be erase after each deployment need to be specify like *volume* (example : `VOLUME /var/html/www/wp-content/uploads`). Be careful !  If repository that content your code source is declared like *volume*, changes files will not be take in charge during redeployment.

In the file `.gitlab-ci.yml`, replace `--useExistingVolumes false` by `--useExistingVolumes true`.

Doing like that, files of your Deployment Server will be kept while, files of your application will be update.

This configuration can be difficult to set up. Do not hesitate to contact us on the [support](https://support.hidora.com/portal/newticket).
If you found erros or optimisations, please tell us on [GitHub](https://github.com/HidoraSwiss/documentation) !

# Deployment with Gitlab and git+ssh

![Deployment with Gitlab and git+ssh](../../../images/PipelineGitlabGitSsh.png)

## Deployment on a Docker node

> These functionality does not work with PaaS nodes which have been created via the marketplace. To deploy these environments, you need to deploy it on a Storage node or configure an [deployment on a node PaaS](https://github.com/jelastic-jps/git-push-deploy).

### Base configuration

This tutoriel consider that you have these next configuration:
- An environment where you want to deploy an application containing a node where you will put your files. We will call this node "Deployment server".
- A gitlab with the appliation source code to deploy (in a repository).

>If the Gitlab is also hosted on Hidora, it is necessary to do few adjustements to make it work. It will be specify on the different steps.

### 1. Generate an SSH Key pair for the Gitlab server

Above all, it is necessary to generate SSH key pair which will allowed Gitlab to push changes on Deployment Server.

For that, use the command line below : 
```bash
ssh-keygen -t rsa -b 4096
```

When it ask you, specify a key name (different of *id_rsa*) proposed by default and
do not enter a password for the key, if not it will be asked during the automation deployment, and will create an error.

After creation of the key, you will have two files (by default, it will be located in `~/.ssh/`)

- `<name-key>` which is the *private key*
- `<name-key>.pub` which is the *public key*

Log in into your Deployment Server via SSH and copy the content of your **public key** following the text in the file `~/.ssh/authorized_keys`. If it does not exist, create it.

### 2. Configuration of the Deployment Server

#### 2.1 Authenticate to Gitlab

Previously, we have create an SSH key pair to allowed Gitlab to connect on the Deployment Server to push changes. But, it is also necessary to authenticate the Deployment Server to Gitlab for allowing it to clone files (which is already in the repository).

For that, generate another SSH key pair, but directly on the Deployment Server : 

```bash
ssh-keygen -t rsa -b 4096
```

When it ask you yo enter a name for your key, **keep the default name** (*id_rsa*) to avoid further configuration.

After, you will have two files:

- `~/.ssh/id_rsa` which is the *private key* for your Deployment Server 
- `~/.ssh/id_rsa.pub` which is *public key* of your Deployment Server

On Gitlab, from the directory that you want to deploy, go into *Settings > Repository > Deploy keys* and add a new public key. The public key of your Deployment Server (copy/past the content in `~/.ssh/id_rsa.pub`), like that, your Deployment Server will be able to retrieve files of your project repository by making a `git clone`.


#### 2.2 Git repository

On the Deployment Server, we will create a repository which will contains the files deployed.
Clone the Gitlab repository by separate files sources and repository which will manage git versions:

```bash
cd / && mkdir git && cd git
git clone --bare <url-du-repo-gitlab> # Which will create repository with the name <bare-dir>
nano <bare-dir>/hooks/post-receive # Check "Content of the file post-receive" below
chmod +x <bare-dir>/hooks/post-receive # We give the execution right to the script post-receive
rm -rf <files-dir>/* # To be sure that the file repository is empty
git clone <bare-dir> <files-dir> # We clone the Bare Repository towards the location of our files
rm -rf <files-dir>/.git # We delete the repository .git because we have already <bare-dir> which manage Work Tree.
```

`<files-dir>` is a path of the directory where you want to put your files deployed. For example, on a *ngingxphp*, it will be the directory `/var/www/webroot/ROOT/` (it depends of your configuration)

>If your Gitlab is hosted on Hidora, during the `git clone`, you need to use the port *22* in the `<url-of-your-gitlab-repository>`

**Content of the file `post-receive`:**

```bash
#!/bin/sh
GIT_WORK_TREE=<files-dir> git checkout -f
```

In this way, for each push in the repository `<bare-dir>`, the script `post-receive` will be launch and will update properly the files deployed  `<files-dir> `.

>Be careful: If there was change changes on the files followed by git on the Deployment Server, changes will be lost during the `git checkout`. Make sure to configure corectly the `.gitignore` to avoid issues.

More information about the deployment with `git checkout` on this [link](http://gitolite.com/deploy.html).

### 3 Gitlab configuration

#### 3.1 Adress to push on the Deployment Server

##### Gitlab hosted outside Hidora

The pipeline used a SSH adress to push changes on the Deployment Server. It is composed like below: 
- The node ID of the Deployment Server, visible on Hidora's web interface.
- L'ID of your Hidora Account, visible on your account's parameters (in ssh tab). For example if you have an adresse like this `ssh 346@gate.hidora.com -p 3022`, your ID is *346*.
- The SSH gateway of Hidora, is fixed to `gate.hidora.com:3022`.
- The **absolute** path towards your repository  `<bare-dir>` on yout Deployment Server.

Your adress of your deployment will be simililar to: 
```
ssh://<node-id>-<user-id>@gate.hidora.com:3022<bare-dir>
```

Here, is an example with real values:

```
ssh://16614-346@gate.hidora.com:3022/git/monsite.git
```

##### Gitlab hosted on Hidora

If your Gitlab is hosted on Hidora, the SSH connection will not go through the SSH gateway. The path will be different, it use:
- The username on the Deployment Server, it is generally `root`.
- The intern IP address (private address) of your Deployment Server, visible on the Hidora's web interface.
- The **aboslute** path toward your repository `<bare-dir>` on your Deployment Server.

Your adress of your deployment will be simililar to: 

```
ssh://root@<ip-interne>/<bare-dir>
```

Here, is an example with real values:

```
ssh://root@10.102.2.54/git/monsite.git
```
Normally, the Firewall blocked SSH connection. To allowed the connection, you need to create an endpoint binding a public port to the port 22 of your Deployment Server. You can do it directly on the settings of your environment, in the *Endpoints* tab. Click on *Add*, select the node of your Deployment Server and anter *22* as private port, and then *Add*.

#### 3.2 Add variables for the pipeline

The pipeline that we will configure will need, at least two variables:

- `SSH_PRIVATE_KEY`: is the content of your **private key ** that we have generated on the first part.
- `DEPLOY_GIT_TEST`: is the path which allowed you to push changes into the git repository on the Deployment Server(3.1).

If you have a *test / pre-prod / production server*, it is also possible to provide a variable  `DEPLOY_GIT_PROD` binding to a correct URL to push on your Production Server. To activate this feature on your Production Server, it will be required to launch it manually (with a click) on Gitlab.

From your project directory that you want to deploy on Gitlab, go to *Settings > CI/CD > Secret Variables* and add the two variables with the correct values. These variables will be used by your Deployment pipeline.

#### 3.3 Ajout du pipeline de déploiement

To configure a Deployment Pipeline into your repository, create a new file call  `.gitlab-ci.yml` with the content below and push on Gitlab with your others files.

```yaml
image: alpine
stages:
  - deploy
before_script:
    # Install SSH client and git
  - apk add --no-cache openssh-client git
    # Init SSH client
  - eval $(ssh-agent -s)
    # Add given SSH private key
  - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add - > /dev/null
    # Prepare SSH configuration
  - mkdir -p ~/.ssh
  - chmod 700 ~/.ssh
  - touch ~/.ssh/known_hosts
  - chmod 644 ~/.ssh/known_hosts
  - cat ~/.ssh/known_hosts
    # Set git variables
  - git config --global user.email "$GITLAB_USER_EMAIL"
  - git config --global user.name "$GITLAB_USER_NAME"

# Deploy to Test env
deployTest:
  only:
    - master
  stage: deploy
  script:
      # Add deploy host to SSH known hosts
    - "HOST=$(echo $DEPLOY_GIT_TEST | awk -F \"(//?)|@\" '{ print $3 }' | awk -F : '$2  ~ /^[0-9]+$/ {print \"-p\",$2,$1;next} {print $1}')"
    - ssh-keyscan $HOST >> ~/.ssh/known_hosts
      # Add new remote repository to local git (replace if exists)
    - git remote rm test || true
    - git remote add test $DEPLOY_GIT_TEST
      # Push branch master to remote server
    - git branch -f master HEAD
    - git push -v test master

# Deploy to Production env
deployProd:
  # Need to be started manually
  when: manual
  only:
    - master
  stage: deploy
  script:
    - "HOST=$(echo $DEPLOY_GIT_PROD | awk -F \"(//?)|@\" '{ print $3 }' | awk -F : '$2  ~ /^[0-9]+$/ {print \"-p\",$2,$1;next} {print $1}')"
    - ssh-keyscan $HOST >> ~/.ssh/known_hosts
    - git remote rm prod || true
    - git remote add prod $DEPLOY_GIT_PROD
    - git branch -f master HEAD
    - git push -v prod master
```

> The last party *Deploy to Production env* is not mandatory if you don't have the need of a Production Server with manual deployment (so you can delete it)

For additional information about this pipeline confguration with Gitlab, have a look on the [documentation of  Gitlab](https://docs.gitlab.com/ce/ci/yaml/README.html).

Each push on Gitlab, will execute actions describe on the pipeline and will frowarding to the Deployment Server.
You can use the functionality of *CI/CD > Pipelines*.

This configuration can be difficult to set up. Do not hesitate to ask questions. 

Did you find an arror or any advice to improve it ? Share it with us on  [GitHub](https://github.com/HidoraSwiss/documentation) !

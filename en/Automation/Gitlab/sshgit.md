# Deployment with Gitlab and git+ssh

![Deployment with Gitlab and git+ssh](../../../images/PipelineGitlabGitSsh.png)

## Deployment on a Docker node

> This functionality does not work with PaaS nodes which have been created via the marketplace. To deploy to these environments, you need to deploy files on a Storage node or configure [deployment on a node PaaS](https://github.com/jelastic-jps/git-push-deploy).

### Base configuration

This tutoriel considers that you have this configuration:
- An environment where you want to deploy an application containing a node where you will put your files. We will call this node "Deployment server".
- A gitlab with the appliation source code to deploy (in a repository).

>If the Gitlab is also hosted on Hidora, it is necessary to do a few adjustements to make it work. It will be specify at each step.

### 1. Generate an SSH Key pair for the Gitlab server

In first place, it is necessary to generate SSH key pair which will allowed Gitlab to push changes on the deployment server.

Use the command line below : 

```bash
ssh-keygen -t rsa -b 4096
```

When asked, specify a key name (different of *id_rsa*) proposed by default and
do not enter a password for the key (if you do, it will be asked during the automation deployment and will create an error).

After creation of the key, you will have two files (by default, it will be located in `~/.ssh/`)

- `<name-key>` which is the *private key*
- `<name-key>.pub` which is the *public key*

Log in into your Deployment Server via SSH and copy the content of your **public key** following the current content in `~/.ssh/authorized_keys`. If it does not exist, create it.

### 2. Configuration of the Deployment Server

#### 2.1 Authenticate to Gitlab

Previously, we have created an SSH key pair to allowed Gitlab to connect to the Deployment Server. But, it is also necessary to authenticate the Deployment Server to Gitlab for allowing it to clone files of the repository.

Generate another SSH key pair directly on the Deployment Server : 

```bash
ssh-keygen -t rsa -b 4096
```

When you're asked to enter a name for your key, **keep the default name** (*id_rsa*) to avoid further configuration.

Then, you will have two files:

- `~/.ssh/id_rsa` which is the *private key* for your Deployment Server 
- `~/.ssh/id_rsa.pub` which is *public key* of your Deployment Server

On Gitlab, from the directory that you want to deploy, go into *Settings > Repository > Deploy keys*, add a new public key and past the content of your public key file (copy/past the content in `~/.ssh/id_rsa.pub`). Now, your Deployment Server will be able to retrieve files of your project repository by making a `git clone`.


#### 2.2 Git repository

On the Deployment Server, we will create a repository which will contains the deployed files.
Clone the Gitlab repository separating the work tree (sources files) and the git configuration (bare repo).

```bash
cd / && mkdir git && cd git
git clone --bare <url-du-repo-gitlab> # Which will create repository with the name <bare-dir>
nano <bare-dir>/hooks/post-receive # Check "Content of the file post-receive" below
chmod +x <bare-dir>/hooks/post-receive # We give the execution right to the script post-receive
rm -rf <files-dir>/* # To be sure that the file repository is empty
git clone <bare-dir> <files-dir> # We clone the Bare Repository towards the location of our files
rm -rf <files-dir>/.git # We delete the repository .git because we already have <bare-dir> which manage the Work Tree.
```

`<files-dir>` is a path of the directory where you want to put your deployed files. For example, on a *ngingxphp*, it will be the directory `/var/www/webroot/ROOT/` (it depends of your configuration)

>If your Gitlab is hosted on Hidora, during the `git clone`, you need to use the port *22* in the `<url-of-your-gitlab-repository>`

**Content of the file `post-receive`:**

```bash
#!/bin/sh
GIT_WORK_TREE=<files-dir> git checkout -f
```

In this way, for each push in the repository `<bare-dir>`, the script `post-receive` will be launch and will update properly the files deployed to `<files-dir> `.

>Be careful: If there was changes on the files followed by Git on the Deployment Server, they will be lost during the `git checkout`. Make sure to configure corectly the `.gitignore` to avoid issues.

More information about the deployment with `git checkout` on this [link](http://gitolite.com/deploy.html).

### 3 Gitlab configuration

#### 3.1 Address to push on the Deployment Server

##### Gitlab hosted outside Hidora

The pipeline uses a SSH address to push changes on the Deployment Server. It is composed as following: 
- The node ID of the Deployment Server, visible on Hidora's web interface.
- The ID of your Hidora Account, visible on your account's parameters (in SSH tab). For example, if you have an address like this `ssh 346@gate.hidora.com -p 3022`, your ID is *346*.
- The SSH gateway of Hidora, fixed to `gate.hidora.com:3022`.
- The **absolute** path to your repository `<bare-dir>` on your Deployment Server.

Your address of your deployment will be simililar to: 
```
ssh://<node-id>-<user-id>@gate.hidora.com:3022<bare-dir>
```

This is an example with real values:

```
ssh://16614-346@gate.hidora.com:3022/git/mysite.git
```

##### Gitlab hosted on Hidora

If your Gitlab is hosted on Hidora, the SSH connection will not go through the SSH gateway. The path will be different:
- The username on the Deployment Server, it is generally `root`.
- The internal IP address (private address) of your Deployment Server, visible on the Hidora's web interface.
- The **aboslute** path to your repository `<bare-dir>` on your Deployment Server.

Your address of your deployment will be simililar to: 

```
ssh://root@<ip-interne>/<bare-dir>
```

Example with real values:

```
ssh://root@10.102.2.54/git/mysite.git
```

Normally, the Firewall blocks SSH connection. To allowed the connection, you need to create an endpoint binding a public port to the port 22 of your Deployment Server. You can do it directly on the settings of your environment, in the *Endpoints* tab. Click on *Add*, select the node of your Deployment Server and enter *22* as private port, and then *Add*.

#### 3.2 Add variables for the pipeline

The pipeline that we will configure needs at least two variables:

- `SSH_PRIVATE_KEY`: content of your **private key ** that we have generated on the first part.
- `DEPLOY_GIT_TEST`: path which allowed you to push changes into the git repository on the Deployment Server(3.1).

If you have a *test / pre-prod / production server*, it is also possible to provide a variable `DEPLOY_GIT_PROD` binding to a correct URL to push on your Production Server. To activate this feature on your Production Server, it will be required to launch it manually (with a click) on Gitlab.

From your project directory that you want to deploy on Gitlab, go to *Settings > CI/CD > Secret Variables* and add the two variables with the correct values. These variables will be used by your Deployment pipeline.

#### 3.3 Add deployment pipeline

To configure a Deployment Pipeline into your repository, create a new file call `.gitlab-ci.yml` with the content below and push on Gitlab with your other files.

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

> The last part called *Deploy to Production env* is not mandatory if you don't have the need of a Production Server with manual deployment (so you can delete it)

For additional information about this pipeline confguration with Gitlab, have a look on the [documentation of  Gitlab](https://docs.gitlab.com/ce/ci/yaml/README.html).

Each push on Gitlab will execute actions described on the pipeline and will forward changes to the Deployment Server.
You can see logs on *CI/CD > Pipelines* on Gitlab.

This configuration can be difficult to set up. Do not hesitate to ask questions. 

Did you find an error or any advice to improve it ? Share it with us on  [GitHub](https://github.com/HidoraSwiss/documentation) !

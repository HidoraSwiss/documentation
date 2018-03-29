# Déploiement avec Gitlab et git+ssh

![Déploiement avec Gitlab et git+ssh](../../../images/PipelineGitlabGitSsh.png)

## Déploiement sur un noeud Docker

> Cela ne fonctionne pas pour les noeuds de type Paas généralement utilisés pour les noeuds principaux d'un environnement par les applications du marketplace. Pour déployer dans ces environnements, il faut soit déployer sur un noeud de type Storage, soit configurer un [déploiement sur un noeud Paas](https://github.com/jelastic-jps/git-push-deploy).

### Configuration de base

Ce tutoriel considère que vous disposez de la configuration minimale suivante:

- Un environnement sur lequel vous souhaitez déployer une application contenant un noeud sur lequel vous voulez placer vos fichier. On appelera ce noeud le "serveur de déploiement".
- Un Gitlab avec les sources de l'application à déployer dans un dépôt.

> Si le Gitlab est aussi hébergé chez Hidora, il est nécessaire de faire quelques adaptions pour que cela fonctionne. Cela sera précisé dans les étapes.

### 1. Générer une paire de clé SSH pour le serveur Gitlab

Avant tout, il est nécessaire de générer une paire de clé SSH qui va permettre à Gitlab de pusher les modifications sur le serveur de déploiement.

Pour cela, utilisez cette commande (peu importe le système où cette commande est lancée, les clés seront copier aux bons endroits plus tard) :

```bash
ssh-keygen -t rsa -b 4096
```

Lorsqu'on vous le demande, préciser un nom de clé (autre que *id_rsa* proposé par défaut) et n'entrez pas de mot de passe pour la clé, sinon il sera demander lors du déploiement automatique et fera planter le pipeline.

Après génération, vous aurez deux fichiers (par défaut, elles se placent dans `~/.ssh/`):

- `<nom-clé>` qui correspond à la *clé privée*
- `<nom-clé>.pub` qui correspond à la *clé publique*

Connectez vous sur le serveur de déploiement par SSH et copier le contenu de votre clé **publique** à la suite du fichier `~/.ssh/authorized_keys`. S'il n'existe pas, créez le.

### 2. Configuration du serveur de déploiement

#### 2.1 Authentification auprès de Gitlab

Précédemment, on a créé une paire de clé pour autoriser Gitlab à se connecter sur le serveur de déploiement pour pousser des modifications. Mais il est également nécessaire d'authentifier le serveur de déploiement auprès de Gitlab pour qu'il puisse cloner les fichiers déjà présents dans le dépôt.

Pour ce faire, générez une deuxième paire de clé, cette fois-ci directement sur le serveur de déploiement:

```bash
ssh-keygen -t rsa -b 4096
```

Lorsqu'on vous le demande, **laissez le nom de clé par défaut** (*id_rsa*) pour éviter des configurations supplémentaires.

Après la génération, vous avez deux fichiers:

- `~/.ssh/id_rsa` qui correspond à la *clé privée* de votre serveur de déploiement
- `~/.ssh/id_rsa.pub` qui correspond à la *clé publique* correspondante

Sur Gitlab, depuis la page de votre dépôt à déployer, rendez vous dans *Settings > Repository > Deploy Keys* et ajoutez une nouvelle clé de déploiement avec le contenu de `~/.ssh/id_rsa.pub`. Ainsi, votre serveur de déploiement aura l'autorisation de récupérer les fichiers du dépôt en faisant un `git clone`.

#### 2.2 Répertoires git

Toujours sur le serveur de déploiement, nous allons créer le dossier qui contiendra les fichiers déployés.
Pour cela, nous allons cloner le répertoire de Gitlab en séparant les fichiers sources et le dossier qui gère les versions avec git:

```bash
cd / && mkdir git && cd git
git clone --bare <url-du-repo-gitlab> # Cela créé un dossier qu'on appellera <bare-dir>
nano <bare-dir>/hooks/post-receive # Voir "Contenu du fichier post-receive" plus bas
chmod +x <bare-dir>/hooks/post-receive # On donne le droit d'exécution au script post-receive
rm -rf <files-dir>/* # On s'assure que le dossier des fichiers est vide
git clone <bare-dir> <files-dir> # On clone le bare repo vers l'emplacement de nos fichiers
rm -rf <files-dir>/.git # On retire le dossier .git car on a déjà le <bare-dir> qui gère ce work tree
```

`<files-dir>` correspond au chemin absolu du dossier où vous souhaitez placer vos fichiers déployés. Par exemple, sur un noeud *nginxphp*, ce sera le dossier `/var/www/webroot/ROOT/`. Cela dépend de votre configuration.

> Si votre Gitlab est hébergé chez Hidora, lors du `git clone`, il faut que le port utilisé dans `<url-du-repo-gitlab>` soit *22*.

**Contenu du fichier `post-receive`:**

```bash
#!/bin/sh
GIT_WORK_TREE=<files-dir> git checkout -f
```

Ainsi, à chaque fois qu'on pushera des modifications dans le dossier `<bare-dir>`, le script `post-receive` sera lancé et mettra à jour proprement le dossier des fichiers déployés.

> Attention: S'il y a eu des modifications sur des fichiers suivis par git sur le serveur de déploiement, les modifications seront perdues lors du `git checkout`. Veillez à bien configurer le `.gitignore` pour éviter des problèmes.

Plus d'infos sur le déploiement avec `git checkout` [sur ce lien](http://gitolite.com/deploy.html).

### 3 Configuration de Gitlab

#### 3.1 Adresse pour pusher sur le serveur de déploiement

##### Gitlab à l'extérieur de Hidora

Le pipeline se sert d'une adresse SSH pour pusher les modifications sur le serveur de déploiement. Elle est formée par :

- le node ID du serveur de déploiement, visible dans l'interface web de Hidora.
- L'ID de votre compte sur Hidora, visible dans les paramètres du compte sur l'interface web, dans l'onglet où il est possible d'ajouter une clé publique. Si vous avez `ssh 346@gate.hidora.com -p 3022`, alors votre ID est *346*.
- Le port SSH et l'URL de la passerelle SSH de Hidora, normalement fixé à `gate.hidora.com:3022`.
- Le chemin **absolu** vers votre dossier `<bare-dir>` sur le serveur de déploiement.

Votre adresse pour le déploiement sera composée comme cela :

```
ssh://<node-id>-<user-id>@gate.hidora.com:3022<bare-dir>
```

Voici un exemple avec des vraies valeurs:

```
ssh://16614-346@gate.hidora.com:3022/git/monsite.git
```

##### Gitlab hébergé chez Hidora

Si votre Gitlab est hébergé sur Hidora, la connexion SSH ne passera pas par la passerelle SSH. Le chemin est donc différent. Il utilise:

- Le nom d'utilisateur sur le serveur de déploiement. C'est généralement `root`.
- L'adresse IP interne (adresse privée) de votre serveur de déploiement, visible dans l'interface web de Hidora.
- Le chemin **absolu** vers votre dossier `<bare-dir>` sur le serveur de déploiement.

Votre adresse pour le déploiement sera composée comme cela :

```
ssh://root@<ip-interne>/<bare-dir>
```

Voici un exemple avec des vraies valeurs :

```
ssh://root@10.102.2.54/git/monsite.git
```

De base, un firewall bloque l'accès SSH. Pour permettre la connexion, il est nécessaire de créer un endpoint associant un port publique au port 22 de votre serveur de déploiement. Cela se fait dans les paramètres de votre environnement, dans l'onglet *Endpoints*. Cliquez sur *Add*, sélectionnez le node correspondant à votre serveur de déploiement et entrez *22* comme port privé puis cliquez sur *Add*.

#### 3.2 Ajout des variables pour le pipeline 

Le pipeline que l'on va configurer a besoin d'au moins deux variables propres à votre projet:

- `SSH_PRIVATE_KEY`: C'est le contenu de la clé **privée** que l'on a généré dans la première partie.
- `DEPLOY_GIT_TEST`: C'est le chemin permettant de pusher des modifications dans le répertoire git sur votre serveur de déploiement (3.1).

Si vous voulez avoir un serveur de test / pre-prod et un serveur de production, il est également possible de fournir une variable `DEPLOY_GIT_PROD` avec le chemin pour pusher sur un serveur de production. Pour activer ce déploiement en production, il sera nécessaire de le lancer manuellement au clic sur un bouton dans Gitlab.

Depuis la page de votre dépôt à déployer sur Gitlab, rendez vous dans *Settings > CI/CD > Secret variables* et ajouter les deux (ou trois) variables avec les bonnes valeurs. Ces variables seront ainsi utilisables par votre pipeline de déploiement.

#### 3.3 Ajout du pipeline de déploiement

Pour configurer un pipeline de déploiement dans votre dépôt, créez un nouveau fichier nommé `.gitlab-ci.yml` avec le contenu suivant et pushez le sur Gitlab avec vos autres fichiers.

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

> La dernière partie *Deploy to Production env* n'est pas nécessaire si vous n'avez pas besoin d'un serveur de production avec déploiement manuel. Vous pouvez dans ce cas la supprimer.

Pour plus d'informations sur la configuration de pipeline avec Gitlab, consultez la [documentation de Gitlab](https://docs.gitlab.com/ce/ci/yaml/README.html).

À chaque fois que vous pusherez sur Gitlab, ce dernier exécutera les actions décrites dans le pipeline et les fera suivre jusqu'au serveur de déploiement.

Vous pouvez visualiser le bon fonctionnement des pipelines dans *CI/CD > Pipelines*.

Cette configuration peut être relativement difficile à mettre en place. N'hésitez pas à contacter le [support de Hidora](https://support.hidora.com/portal/newticket) pour toute question.

Vous avez trouvé des erreurs ou des optimisations possibles ? Dites le nous sur [GitHub](https://github.com/HidoraSwiss/documentation) !

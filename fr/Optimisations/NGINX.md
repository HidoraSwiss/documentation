# Optimisations pour NGINX

Ces optimisations peuvent être utilisées dans la plupart des environnements du Marketplace utilisant des noeuds NGINX.

## Configuration custom répliquée

Il est possible de modifier la configuration d'un noeud NGINX dans un environnement assez facilement. Cependant, si le noeud NGINX est multiplié (scaling horizontal), cette configuration ne sera pas appliquée sur les noeuds nouvellement créés.

Pour avoir un scaling horizonale propre, il est conseillé d'exporter la configuration de NGINX vers une instance de storage et de mapper le dossier `/etc/nginx` des noeuds NGINX vers cette instance.

1. Créer une instance de Storage. Dans la plupart des environnements clusterisés, cela existe déjà.
2. Déplacer la configuration de NGINX vers l'instance de Storage.
   - Sur un noeud NGINX : `cd /etc/nginx && tar cfz config-nginx.tar.gz`
   - Télécharger le fichier *config-nginx.tar.gz* et l'uploader sur l'instance de Storage (dans le dossier `/tmp` par exemple)
   - Sur le noeud Storage : `mkdir -p /config/nginx && cd /config/nginx && tar xvf /tmp/config-nginx.tar.gz`
3. Mapper le dossier `/etc/nginx/` des noeuds NGINX sur le dossier `/config/nginx` du noeud Storage dans la topologie de l'environnement.
4. Redémarrer les noeuds NGINX pour s'assurer qu'ils prennent la bonne configuration

Ainsi, chaque nouveau noeud NGINX prendra sa configuration dans le dossier `/config/nginx` sur le noeud Storage.
# Optimisations for NGINX

These optimisations can be used in most environments on the Marketplace using nodes NGINX.

## Custom configuration 

It is possible to modify configuration of NGINX node in an anvironmnet very easily.However, if the NGINX node is duplicated (horizontal scaling), this configuration will not be assigned to the new nodes.

For having a clean horizontal scaling, it is advisable to export NGINX configuration into a storage instance and to map the directory `/etc/nginx` of NGNX nodes to it.
1. Create a Storage instance. In the most clusters environment, it already exists.
2.  Move NGINX configuration into Storage instance
    - On NGINX node : `cd /etc/nginx && tar cfz config-nginx.tar.gz`
    - Donwload the file *config-nginx.tar.gz* and upload it on the instance Storage (in the directory `/tmp` for example ).
    - on Storage node : `mkdir -p /config/nginx && cd /config/nginx && tar xvf /tmp/config-nginx.tar.gz`
3. Map the directory  `/etc/nginx/` of NGINX nodes on the Sotrage node `/config/nginx`.
4. Reboot NGINX nodes to be sure that all NGINX nodes have the good configuration.

In this maner, each new NGINX nodes will have its confuration in the directory `/config/nginx` of the Storage instance.


# Optimisations pour cluster Wordpress

## Erreur avec *zero dates*

Selon la configuration MySQL et la version de Wordpress, il peut arriver que vous ne puissiez pas créer de nouveau poste du côté de l'admin car la base de données n'accepte pas les dates où tous les champs sont à 0, ce que fourni Wordpress de manière temporaire.

Pour résoudre cela, il faut retirer deux variables des "modes" de MySQL. **Sur chacun des noeuds MySQL** de votre environnement, réaliser les actions suivantes :

1. Se connecter sur PHPMyAdmin avec les identifiants fournis par e-mail
2. Se rendre sur l'onglet *Variables* depuis la page d'accueil
3. Chercher la variable nommer `sql_mode` et cliquer sur *Éditer*
4. Retirer `NO_ZERO_IN_DATE,NO_ZERO_DATE,` dans les valeurs et cliquer sur *Enregistrer*

Vous pouvez désormais ajouter des postes dans l'admin de Wordpress.

> Pour plus d'infos sur le bug : https://de-ch.wordpress.org/plugins/incorrect-datetime-bug-plugin-fix/
---

copyright:
  years: 2017
lastupdated: "2017-10-18"

---

{:shortdesc: .shortdesc}
{:new_window: target="_blank"}
{:codeblock: .codeblock}
{:pre: .pre}
{:screen: .screen}
{:tip: .tip}
{:download: .download}

# Synchroniser la configuration HA
Deux unités VRA (Virtual Router Appliance) dans une paire HA doivent avoir une configuration suffisamment synchronisée pour que les deux unités se comportent de la même manière. Cette synchronisation s'effectue via des mappes de synchronisation de configuration (`configuration sync-maps`) et vous pouvez sélectionner les parties de la configuration à synchroniser. Si vous effectuez une modification sur une machine, la configuration marquée sera envoyée à l'autre unité.

**REMARQUE :** Cela ne signifie pas que la configuration sur l'autre unité a été sauvegardée. Seules les modifications apportées à la configuration en cours sont effectuées.

La configuration unique à un système ne doit pas être synchronisée sur l'autre système. Des adresses IP et MAC réelles ne doivent pas être synchronisées, par exemple. Le noeud de configuration `system config-sync` même et le noeud `service https` ne peuvent pas du tout être synchronisés.

L'exemple suivant illustre la synchronisation de la configuration (config-sync) :

```
set system config-sync sync-map TEST rule 2 action include
set system config-sync sync-map TEST rule 2 location security firewall
set system config-sync remote-router 192.168.1.22 username vyatta
set system config-sync remote-router 192.168.1.22 password xxxxxx
set system config-sync remote-router 192.168.1.22 sync-map TEST
```

Les deux premières lignes créent la mappe de synchronisation (sync-map) réelle. Ici, la strophe de configuration pour le pare-feu de sécurité (`security firewall`) sera définie dans `sync-map`. Par conséquent, tout changement effectué dans le noeud config sera envoyé à l'unité distante. Toutefois, les changements effectués pour `security user` ne seront pas envoyés, car ça ne correspond pas à la règle. Vous pouvez rendre la mappe de synchronisation `sync-map` aussi spécifique ou générale que vous le souhaitez.

Les trois lignes suivantes désignent l'utilisateur et le mot de passe, l'adresse IP du routeur distant, ainsi que la mappe de configuration (sync-map) à envoyer pour `config-sync`. Tous les changements correspondant aux règles de `TEST`, seront envoyés à `remote-router 192.168.1.22`, en utilisant ces informations de connexion. Notez qu'un appel `REST` est effectué pour exécuter cette opération à l'aide de l'API VRA, donc le serveur HTTPS doit être en cours d'exécution (et non bloqué) sur le routeur distant.

La synchronisation de la configuration (config-sync) se produit lorsque vous validez un changement, donc soyez attentif aux messages d'erreur en provenance de l'unité distante. Si la configuration n'est pas synchronisée, vous devez la corriger sur l'unité distante pour qu'elle soit à nouveau opérationnelle.

Vous pouvez également voir les différences de configuration à l'aide de la commande `show config-sync difference`.

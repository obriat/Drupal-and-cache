DRUPAL & CACHE
==============
Lisbon 2018




# à ajouter :
examples use case
https://www.drupal.org/project/drupal/issues/2145751 Introduce ENTITY_TYPE_list:BUNDLE cache tag
 \Drupal::entityTypeManager()->getDefinition('taxonomy_vocabulary')->getListCacheTags() => config:taxonomy_vocabulary_list
 \Drupal::entityTypeManager()->getDefinition('taxonomy_term')->getListCacheTags() => taxonomy_term_list


# 0
Hi, I'm going to present Drupal and its caching system

# 1
My name is Olivier Briat, 
I started using Drupal about 6 years ago when migrating the Paris national Opera website.
I now live in Nantes (France) where I joined Cap Gemini a major IT consulting company.
I'm a Drupal lead developer, I work mainly on SNCF web sites (the french national railroad company): www.sncf.com, oui.sncf, izy.com, ...

#3
This presentation is mainly based on the one that Berdir did last year at Seville, if you attended it you will not learn much more from this one.
Nonetheless, I try give it a personal touch and a different point of view: I will talks about browser caching and I added some coding use cases and examples.

The other sources of inspiration are:

    - The Render API & Cache API presentation by Artusamak at Drupal Camp Nantes 2016
    - The cache chapter of the Drupal 8 Caching - A Developer’s Guide by  Peter Sawczynec

#4 Topics

    - Caches upstream of Drupal
      - Browser
      - Network caches
    - Internal Page Cache (aka "Anonymous cache")
    - Dynamic Page Caching
    - Render API cache system
    - Using Drupal cache mechanisms in your custom code
    - Cache storage
    - Questions ?

I'll start upstream of Drupal with browser cache.
Then we will come closer to Drupal with proxies & varnish.
We will reach Drupal and talk about the two main Drupal's caching modules : first "Anonymous" page caching, then Dynamic Page Caching
I will give more details about the mecanisms behind it: the render API and cache metadata.
I'll finish with how to use cache in your custom code, with example and use cases.
I'll give a quick overview about bin & backends 

#5 
What'is cache
Store the result of slow calculations so that they do not need to be executed again and so improve performances.

#7
This subject could be quite complexe, so I will stick to a simple example :

When you access a web page for the first time, your browser sends a simple HTTP GET request to the webserver plus a fewer headers.
Response code and headers will be analysed to determine how to cache it, the url will be used as the cache key.
The most common case is a 200 (OK) code and a cache-control header with a max-age that tells on long the browser will have to keep the response (content + headers) before asking for an update. Otherwise the cache-control is set to no-cache or no-store.


After this max-age the browser will send a request with some checksum headers (etags, last-modified)
If they don't match the server will send a brand new response with new header, 
It they match it will just a 304 response code and a few headers.

The vary header expands the cache key, it indicate that there is multiple representation for this content acording to request parameters: User-Agent, accept-encoding, accept-language


So if max-age is set to 3600, each user will only access a webpage once per hour, even if the content has changed meanwhile.


Ok but what about Drupal ?

#8
The cache-control max-age is setup in the Drupal backoffice, try to define a setting that reflect your website lifecycle: news site, low volume blog, .... 

If you need more control on headers, have a look at the HTTP Cache Control module
https://www.drupal.org/project/http_response_headers
https://www.drupal.org/project/http_cache_control
https://www.varnish-software.com/wiki/start/httpcaching_basics.html

#9
For request that are not Response Object (i.e. not managed by Drupal), cache expiration headers are handle by the webserver according to the .htaccess file at project root dir.


#11

You just ahve to copy sites/example.settings.local.php into sites/default/settings.local.php

Let's have a look at its content 

#12
Let's have a look at "settings dot local dot php" file content:
It load developpement services, this files is one directory above.
I'll explains later what exactly is a "cache bin"


TODO  vérifier debug_cacheability_headers

 


https://www.drupal.org/project/redis/issues/2877893#comment-12539462

ISSUE : Bubbling of elements' max-age to the page's headers and the page cache
https://www.drupal.org/project/drupal/issues/2352009

#15

Between you browser and Drupal the request will cross many networks nodes, some of them could add aditional caches.

The most common ones are proxies, they are usually placed at gateways : ISP, company or organisation internet access, ...
They work just like browser cache but at larger scale, they can be targeted with specific cache headers s-max-age,...
A Content Delivery Network is a kind of proxy but focused on performance. Usually it cache big asset files and replicate them around the world so they could be the closest to the users. These network of proxy are proviede as service by comany as Akamaï or CloudFlare.

The last step before Drupal is Varnish.

Varnish is Drupal best buddy, you will find it in front of most of the Drupal large web site, it's a classical choice.
It w

    Couteau suisse des headers http
    Peut cacher un site indisponible
    Peut être piloté par Drupal (module purge)
    Peut-être remplacé par le micro caching de votre serveur Nginx
    Il peut remplacer le module Internal Page Cache

Proxy cache-control: public/private/proxy-revalidate/s-max-age/...
CDN : Pour les assets (fichiers statiques css, img, js,...), proximité de l'utilisateur
Transition "Internal Page Cache"
Mais avant d'aller plus loin

Désactiver le cache et activer les options de débogage.
settings.local.php

$settings['container_yamls'][] = DRUPAL_ROOT . '/sites/development.services.yml';

$config['system.logging']['error_level'] = 'verbose';

$config['system.performance']['css']['preprocess'] = FALSE;
$config['system.performance']['js']['preprocess'] = FALSE;

# $settings['cache']['bins']['render'] = 'cache.backend.null';

# $settings['cache']['bins']['dynamic_page_cache'] = 'cache.backend.null';

development.services.yml

parameters:
  http.response.debug_cacheability_headers: true
services:
  cache.backend.null:
    class: Drupal\Core\Cache\NullBackendFactory

http.response.debug_cacheability_headers permet de faire apparaître les headers HTTP X-Drupal- très utiles pour le débogage du cache Drupal.

Avec drupal console : drupal site:mode dev (https://www.drupal.org/node/2598914)


Mais surtout :

N'oubliez de réactiver les caches avant de tester votre dev...
Cache anonyme de page

    Géré par le module Internal Page Cache
    Cache l'intégralité du rendu html d'une page anonyme dans le cache
    Très rapide
    La clé est l'URL complète
    Utilise les "cache tags", mais pas les "cache contexts" ou le "max age"
    Respect le header "Expires" de la response)
    Comme on l'a vu, il peut être désactivé si on utilise un type de cache similaire (Varnish)
    Header HTTP correspondant : X-Drupal-Cache (HIT/MISS)

On reviendra sur les concepts de meta données de cache : "Cache tags", "cache contexts" ou "max age"
Cache dynamique de page

    Module du coeur : "Dynamic Page Cache"
    Ce cache est plus fin (mais plus lent) que le cache anonyme.
    Il utilise le cache de chaque élément* qui compose une page.
    * On reviendra plus tard sur ce qui compose ces éléments.
    Chaque élément à ses propres méta-données de cache, celles-ci permettent d'en déterminer la validité.
    Elles bouillonnent" et sont agrégées aux éléments parents jusqu'à remonter au niveau de la page.
    Donc si le cache d'un élément n'est plus valide, par "bouillonnement", il invalidera donc celui de tous ses parents.
    Voici un exemple :

Module à laisser toujours activé
Schémas à suivre

Voici une page composé de deux blocs avec chacun un liste de noeuds.

Schémas @berdir
Version simplifié des méta données : notamment tags
On note l'héritage

Modification du node 5 et création du node 7 (en rouge les méta-données invalidées)
On note que la création d'un node sport invalide le tag node:sport

En rouge les contenus recalculés :
Nouveau article 7, modification de article 5, article 6 inchangé, zone sport modifié et par héritage la page également. La zone buisness n'est pas recalculé.
Qu'est ce qui se passe si on a une zone avec un forte cardinalité ?
#lazy_builder
et
Auto-placeholdering

Pour empêcher les éléments très dynamiques d'invalider systématiquement le cache de page, ceux-ci sont remplacés par des placeholders, en toute fin de rendu ils sont substitués par le contenu du callback.

https://www.drupal.org/docs/8/api/render-api/auto-placeholdering
Coeur du module BigPipe

Il peut être défini avec le paramètre #lazy_builder

// Callback : class:method ou (mieux) service:method
return [
  '#lazy_builder' => ['hello_world.lazy_builder:renderSalutation', []],
  '#create_placeholder' => TRUE,
];
  

ou être détecté automatique en fonction des condition suivantes (par defaut) :

# Conditions pour être "autoplaceholderer"
  renderer.config:
    auto_placeholder_conditions:
      max-age: 0
      contexts: ['session', 'user']
      tags: []

Conditions facilement modifiables.
Cache de "rendu"

Les éléments utilisés par le "cache dynamique de page" sont bien entendu les "render array" Drupal.

La plupart ont des méta-données de cache (clé #cache) dont les parents héritent :
Fils
Parent
Render array vu par PhpStorm Xdebug

Ces métas-données s'agrègent donc jusqu'à la page et ses headers

Module renderviz : pour le débogage en profondeur des méta-données de cache.
On retrouve nos header http X-Drupal-cache-*
Les méta-données de cache en détails

    Cache Contexts : Permet d'avoir des variantes de cache en fonction du... contexte (theme, language, user roles, permissions, URL, QS, timezone,... )
    Cache Tags : Permet d'invalider des types de caches (node:x, config:, user:x, library_info, route_match, node_list , ...)
    Max Age : Permanent (-1), age en seconds (3600), ne pas cacher (0)

On verra plus loin comment sont calculer les checksum des tags.
Attention à bien vider les caches quand on ajoute on renomme un tag associé à un contenu.

Gérer les méta-données d'une Render Array

$config = \Drupal::config('system.site');
$current_user = \Drupal::currentUser();

$build = [
  '#markup' => t('Salut, %name, bienvenu sur @site!', [
    '%name' => $current_user->getUsername(),
    '@site' => $config->get('name'),
  ]),
  '#cache' => [
    'contexts' => [
      'user', // Sera traiter en #lazybuilding
    ],
    'tags' => $config->getCacheTags(), // ou addCacheableDependency()
  ],
];

// Autre moyen d'ajouter une dépendance de cache.
$renderer = \Drupal::service('renderer');
$renderer->addCacheableDependency(
  $build,
  \Drupal\user\Entity\User::load($current_user->id())
);
// Fusionner des cache tags
$tags = Cache::mergeTags($conf_one->getCacheTags(),$conf_two->getCacheTags());

Récupération d'objets configuration et user
Création d'une render array qui utilise ce objets et dont l'affchage en dépend
Plusieurs manière d'ajouter une dépendance, directement dans le tableau : à la main, avec la méthode getCacheTags.
On après coup avec la méthode addCacheableDependency du service renderer.
Enfin d'autre méthode permettre de fusionner des méta-données de cache.
Attention à bien vider les caches quand on ajoute on renomme un tag associé à un contenu.
Plugin contexts

Fonctionne avec les annotation context (typiquement dans la définition d'un block)

*   context = {
*     "node" = @ContextDefinition("entity:node", label = @Translation("Node"))
*   }

https://api.drupal.org/api/drupal/core!core.api.php/group/annotation/8.5.x
https://api.drupal.org/api/drupal/core!lib!Drupal!Core!Annotation!ContextDefinition.php/group/plugin_context/8.5.x

Les plugin (et donc les blocs) implémentent CacheableDependencyInterface et donc des méthodes pour initialiser les méta-données de cache.

\Drupal\Core\Plugin\ContextAwarePluginBase::getCacheContexts()

  public function getCacheContexts() {
    return Cache::mergeContexts(parent::getCacheContexts(), ['user.node_grants:view']);
  }

  public function getCacheTags() {
    return Cache::mergeTags(parent::getCacheTags(), ['node_list']);
  }

    public function getCacheMaxAge() {
    return 0;
  }

A propos des block, attention à bien rendre content, dans le fichier Twig du contenu, car c'est lui qui porte les méta-données de cache, voir :
https://www.previousnext.com.au/blog/ensuring-drupal-8-block-cache-tags-bubble-up-page
Utilisation du cache dans son code

Utilisateurs, sites-builders, pas de panique :

Drupal gère le cache tout seul !

Développeurs de tous horizons, voici comment l'utiliser :
Static

    "cache" utilisant le fait qu'une variable static n'est pas détruite à la fin de l'exécution d'une fonction.
    On peut continuer à utiliser drupal_static() pour le code procédural.
    En mode OO, n'oubliez pas que les "services" sont des "singletons" et donc leurs propriétés sont également persistantes

Get / Set

function mes_donnees_metier();
  // Ma clé
  $key = 'mon_module' . ':' . __FUNCTION__ . ':' .
  \Drupal::languageManager()->getCurrentLanguage()->getId();
  // Est-ce que le cache existe ?
  if ($cache = \Drupal::cache()->get($key)) {
    $data = $cache->data;
  }
  // Pas de cache : on traite les données et
  // on les stocke dans le cache.
  else {
    $data = mon_traitement_metier_super_lent();
    \Drupal::cache()->set($key, $data);
  }
  return $data;
}

Duration

// Possible de préciser une durée de validité (timestamp)
\Drupal::cache()->set($key, $data, REQUEST_TIME + 600);

On utilise les deux points (:) comme séparateurs hiérarchique.

Tags

// Associer des tags
\Drupal::cache()->set(
  $key,
  $data,
  Cache::PERMANENT,
  [
   'tag1',
    'node:1',
    'config:system.menu',
    'config:mon_module',
  ]
);

// Découvrir les tags des entités :
$node->getCacheTags();
$entity_type->getListCacheTags(); //EntityTypeInterface
\Drupal\views\Entity\View::load('front')->getCacheTags();
;

Suppression / Invalidation

// Suppression du cache (rapide)
\Drupal::cache()->delete('mon_module:donnees_caches');
\Drupal::cache()->deleteMultiple([
  'mon_module:cle1',
  'mon_module:cle2',
  ...
]);
\Drupal::cache()->deleteAll();

// Invalidation : non conseillé (Berdir)

// Invalidation de tag (incrémentation du compteur d'invalidation)
$cache_tag_invalidator->invalidateTags(['my_tag']);
Cache::invalidateTags(['my_tag']);
            

À l'enregistrement d'un cache on stocke de la somme des compteurs d'invalidation des tags concernés.

À l'appel du cache on recalcule ce checksum, s'il diffère le cache est régénéré.
Chaque tag se voit attribuer un compteur qui est incrémenté à chacune de ses invalidations. Le checksum du contenu est simplement la somme de ces compteurs.

Multiples

\Drupal::cache()->getMultiple($keys);
\Drupal::cache()->setMultiple($items);

Utiliser du cache périmé (pour faire patienter)

$cache = \Drupal::cache()->get('my-key', TRUE);
if ($cache && $cache->valid) {
  return $cache->data;
}
elseif (\Drupal::lock()->acquire('my-key')) {
  // Rebuild and set new data.
}
elseif ($cache) {
  // Someone else is rebuilding, work with stale data.
  return $cache->data;
}
else {
  // Wait or rebuild.
}

Bins

Il y a différents "bacs" (bin) de cache :

    bootstrap : pour le démarrage de Drupal.
    render : cache des rendu HTML
    default : cache par défaut, pour de petits contenu avec peu de clés
    data : pour les gros caches avec de nombreuses clés
    discovery : Petit, utilisé fréquemment, principalement par les plugins et les processus de découvertes

On peut créer son propre "bin". On peut associer un backend différent à chaque "bin".
bootstrap est rarement invalidé.
Oui d'ailleurs c'est quoi un backend ?
Backends

Méthode de stockage d'un bin.

    Dans le core core/core.services.yml :
        Memory : En mémoire, donc non persistant
        Database : Base de donnée, le stockage par défaut *
        APCu : Partage mémoire au travers du processus PHP (donc pas de partage avec drush/CLI ou avec d'autres frontaux)
        Null : pour désactiver le cache (dev)

    Module contribués :
        Slushi Cache : Idem database, mais avec une durée de vie paramétrable
        Memcache : Base clé/valeur stocké en RAM cache.backend.memcache_storage
        Redis: Base clé/valeur stocké en mémoire cache.backend.redis

Le ChainedFast Backend (cache.backend.chainedfast) permet de chaîner un backend rapide au-dessus d'un backend lent.

* Attention, une partie du cache n'est pas détruite mais invalidé, donc les tables de cache peuvent devenir conséquentes.
Merci
à toute l'équipe du DrupalCamp Lannion 2017
aux sponsors
Des questions ?
Références
## Conférences - https://md-systems.github.io/drupal-8-caching/ - https://nantes2016.drupalcamp.fr/programme/sessions/render-api-cache-api - https://pnwdrupalsummit.org/sites/default/files/slides/Drupal%208%20Caching.pdf - Drupal 8 cache for developers - José Jiménez Carrión #DrupalCampES @jjca : https://youtu.be/kfy_JAKudnw - Drupal 8 Caching: A Developer’s Guide : https://youtu.be/eB4NWo5XwMY - BigPipe : https://youtu.be/JwzX0Qv6u3A
## APIs et docs - https://api.drupal.org/api/drupal/core!core.api.php/group/cache/8.5.x - https://www.drupal.org/docs/8/administering-drupal-8-site/internal-page-cache - https://www.drupal.org/docs/8/core/modules/dynamic-page-cache/overview - https://www.drupal.org/docs/8/api/render-api/auto-placeholdering - https://www.drupal.org/docs/8/api/render-api/cacheability-of-render-arrays - https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching - https://www.drupal.org/project/renderviz
Livre

    Drupal 8 Module development de Daniel Sipos (upchuk)
    seulement 5€ chez Packt : https://www.packtpub.com/web-development/drupal-8-module-development

## Serveurs - https://varnish-cache.org - https://redis.io/documentation

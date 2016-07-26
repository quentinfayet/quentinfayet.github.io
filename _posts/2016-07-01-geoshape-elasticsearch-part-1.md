---
layout: page
title: Les Geoshapes avec Elasticsearch (1/2)
comments: false
modified: 2016-07-01
tags: ["elasticsearch", "geoshape", "geolocation"]
---


Dans un article précédent, j'avais parlé des geopoints, qui permettent d'indexer des positions géographiques. Indexer un point c'est bien joli, mais en général, moi, depuis la maternelle, je relie des points pour faire des figures. Et heureusement, c'est visiblement aussi le cas des dev' d'Elasticsearch, car ils ont prévu, pour indexer des formes complexes, un type particulier : La geoshape !
Dans ce premier de deux articles, je vais vous présenter succinctement les tenants et les aboutissants de la geoshape, la théorie qui fait tourner le truc, ainsi que l'indexation des geoshapes. Dans un autre article, afin de rester dans des temps de lecture raisonnables, je vous présenterai les différentes queries qui vont avec les geoshapes.

Afin que vous puissiez tester les exemples, je vous fournis un cluster Elasticsearch prêt à l'emploi, sous Docker (ce qui sous-entend que vous avez Docker et docker-compose d'installé sur votre machine) : https://github.com/quentinfayet/elasticsearch-geoshape.

Afin de lancer le cluster, une simple commande :

```
docker-compose up
```

# Geohash VS Quadtree

Vouloir indexer des formes géographiques complexes, c'est bien beau. Mais quand on sort du simple rectangle pour se diriger vers des formes polygonales, là, l'affaire se corse un peu.

Aussi, Elasticsearch vous offre le choix du comment vous allez stocker vos geoshapes : Le *Geohash* ou le *Quadtree*.

## Le Geohash

Si vous avez lu l'article précédent, alors vous connaissez déjà notre pote le geohash. Mais pour vous, au fond de la classe, je rééxplique brièvement le concept.

Le geohash divise la Terre en un quadrillage plus ou moins resserré. Chaque "case" du quadrillage est une *cellule*, et chaque *cellule* a son propre petit nom, à savoir son *geohash*. Chaque cellule se subdivise ensuite en d'autres cellules plus petites, qui ont un geohash dérivant de leur cellule parente. Par exemple, si l'on considère le *geohash* "u53", alors la cellule parente est "u5", et la cellule parente de u5 est "u".

Mais le *geohash* souffre d'un énorme problème. Comme je vous l'expliquais, le propre de cet algorithme est de produire un quadrillage, où chaque cellule est elle-même subdivisée, et ainsi de suite. Mais considérons l'image suivante :

![les limites du *geohash*](/images/geoshape-elasticsearch-1/geohash-limits.png)

Comme vous pouvez le constater, la France est ici divisée en 2 cellules de geohash : "gb" et "u0". Je vous ai expliqué précédemment que, notamment, le geohash pouvait parfois servir pour déterminer la proximité de deux lieux : plus le préfixe commun du geohash de deux lieux est long, plus les deux lieux sont proches l'un de l'autre... En théorie. Sauf s'ils se trouvent dans deux cellules différentes, comme c'est par exemple ici le cas pour Nantes et Tours : deux villes géographiquement proches à l'échelle de la planète, mais n'ayant pas l'once d'un préfixe de geohash en commun. Et si vous "zoomez" encore un peu plus, en vous plongeant plus profondément dans les méandres du geohash, vous vous rendrez vite compte que deux lieux séparés par seulement quelques mètres, quelques centimètres, quelques millimètres seulement peuvent avoir des geohashs complètement opposés.

L'une des solutions à ce problème est, lorsque l'on cherche à estimer la proximité géographique de deux lieux, de considérer non pas uniquement les cellules respectives de ces deux lieux, mais aussi leurs 8 cellules voisines (une pour chaque côté de la cellule, plus une pour chaque coin).

## Le Quadtree

Le Quadtree lui est le petit nouveau de la bande. En gros, son nom indique assez bien ce qu'il fait. Vous prenez une forme (pour faire simple, un carré), vous le subdivisez en 4 autres carrés (d'où le "quad"). Vous répétez l'opération pour chaque carré ainsi obtenu, encore et encore.

Le résultat est une structure de type arbre, (d'où le "tree"), qui ressemble à cela :

![Le quadtree](/images/geoshape-elasticsearch-1/quadtree.png)

Dans le cas d'Elasticsearch, le quadtree en question est rectangulaire. Dans le cas pratique que j'ai concocté, je vous montrerai comment configurer le quatree. Au niveau d'Elasticsearch, et plus précisément d'Apache Lucene, le stockage d'un quadtree est une suite de bits, ce qui revient à dire que c'est une chaîne de caractères.

## Au final, geohash ou quadtree ?

Ce n'est pas une question à laquelle la réponse est binaire. En effet, si les avantages de l'un compensaient vraiment ses défauts, alors Elasticsearch aurait déjà probablement fait le choix pour vous.

Je ne sais pas si vous avez remarqué, mais le principe qui se cache aussi bien sous le geohash ou sous le quadtree est sensiblement le même : Ce sont des arbres.

Le geohash est un arbre dont chaque noeud se divise en 32 branches, et le quadtree est un arbre dont les noeuds se divisent en 4 branches (ce qui ne le rend pas moins imprécis, tout dépend de la profondeur de l'arbre).

Geohash a un petit avantage sur le quadtree : il est intégré dans beaucoup de services tiers. Il existe beaucoup de services qui peuvent construire des maps, calculer des distances et d'autres valeurs, et qui acceptent les geohashs en entrée.

En revanche, sachez que depuis les versions récentes d'Elasticsearch, la performance du quadtree a été bien améliorée.

Parce qu'en réalité, lorsque que vous indexerez un polygone (disons un triangle) dans Elasticsearch, ce sera en réalité les noeuds du geohash ou du quadtree inclus dans le polygone qui seront indexés ; ainsi, de la profondeur de vos arbres dépendra la "résolution" de votre forme, et la précision avec laquelle vous pourrez faire certaines requêtes, comme par exemple des requêtes d'intersection. Prenons un exemple, voyons comment serait indexé un triangle (tracé en rouge), dans deux arbres de profondeur différente :

| ![triangle low definition](/images/geoshape-elasticsearch-1/triangle_low.png) | ![triangle high definition](/images/geoshape-elasticsearch-1/triangle_high.png) |

Sur l'image de gauche, notre triangle est en réalité constitué des cases colorées en bleu. Finalement, ce n'est qu'une représentation grossière du triangle ; alors que sur l'image de droite, vous remarquez que la grille est bien plus "fine" (l'arbre a une profondeur plus élevée), et la représentation de notre triangle est bien meilleure.
Donc, à la question "geohash ou quadtree", je ne peux pas répondre de manière tranchée. Chaque cas est spécifique, et il vous faut trouver le meilleur compromis entre le type d'arbre (geohash / quadtree), la profondeur à donner (la "résolution" de vos shapes), et les performances que vous voulez atteindre. Car bien évidemment, une profondeur élevée vous donnera une forme très détaillée, au dépit de performances dégradées.

# Indexer une geoshape

Bon, après ce discours bien barbant sur le pourquoi du comment des quadtree et des geohashs, je vous propose de se salir un peu les mains, et de regarder de plus près comment travailler avec les geoshapes.

## Le mapping

Lorsque l'on travaille avec Elasticsearch, la première chose sur laquelle l'on se penche quand on veut indexer des informations, c'est le mapping.
Le mapping de la geoshape ressemble quelque peu au mapping du geopoint, que vous pouvez retrouver dans mon précédent article.

La première chose à faire est de décider si l'on va utiliser des geohashs ou un quadtree pour le calcul des points de notre geoshape ; ici, il s'agit du paramètre `tree` de notre mapping. Ce paramètre définit avec quelle méthode (geohash ou quadtree) le PrefixTree de chaque point sera construit.

Avec le paramètre `tree` viennent deux autres paramètres. Le premier est `precision`, il défini la précision du PrefixTree, c'est à dire, grosso-modo, la "marge d'erreur" que vous autorisez pour chaque point qui compose votre geoshape. Il s'agit d'un nombre entier, optionnellement suivi par une unité de distance (par exemple: `1km` pour une précision de 1 kilomètre, ce qui, ramené à l'échelle de la planète, n'est pas mal), ou bien encore `1mm` pour une précision d'un millimètre, (là, c'est carrément abusé). Le deuxième paramètre est `tree_levels`, que vous pouvez définir si vous souhaitez vous passer de `precision`. Il s'agit de la profondeur de votre arbre ; plus votre arbre est profond, plus le placement des points de la geoshape est précis.

Vous pouvez donc définir soit une `precision`, soit un `tree_levels`, à vous de choisir selon vos besoins.

Attention toute fois à ne pas tomber dans le piège trop tentant du "Wesh, j'mets une précision d'1mm, au moins j'aurais pas de problèmes, je serais hyper précis, swag absolu". Oui... mais non. Un peu de bon sens suffit pour se rendre compte que plus la précision est élevée / plus la profondeur de l'arbre est grande, plus l'indexation de la geoshape est volumineuse. Sur le cluster de ton Raspberry où tu mets 50 geoshapes, tu t'en fous... Mais sur un cluster de production où se côtoient des millions de geoshapes, et le contrôle de la volumétrie ne doit pas vous échapper et reste un point important, un calibrage pointu de la profondeur de l'arbre ou de la précision peut vous éviter un cluster qui s'écroule à 3h du mat'...

Une autre option, apparue avec Elasticsearch 2, est la `strategy`. C'est une option importante dans le sens où elle va définir non seulement la manière dont la geoshape sera décomposée dans votre arbre, mais également les possibilités que vous aurez ensuite en termes de queries. Le champ `strategy` peut prendre deux valeurs. La première, term ne permet que l'indexation de points, et ne supporte qu'un seul des trois types de requêtes possibles sur les geoshapes. La deuxième valeur possible est `recursive`, qui elle permet l'indexation de tout type de geoshape, et la possibilité de performer tous les types de queries geospatiales sur les geoshapes de votre index.

Bon, maintenant, jetons un oeil au mapping final de l'exemple que je vous ai préparé :

```
{
  "mappings": {
    "shape": {
      "properties": {
        "polygon": {
          "type": "geo_shape",
          "tree": "quadtree",
          "strategy": "recursive",
          "precision": "1km"
        }
      }
    }
  }
}
```

J'ai donc créé un mapping pour un type nommé `shape`, qui contient un champ `polygon` qui est notre geoshape. J'ai donc utilisé un tree de type quadtree (car nous avions déjà vu le geohash dans l'article sur les geopoints), en paramètrant une `precision` de 1km.

Il ne nous reste plus qu'à mettre le mapping en place, dans un index que nous appellerons `map`.

Le mapping est disponible à la racine du repository Github que je vous ai fournis (https://github.com/quentinfayet/elasticsearch-geoshape), dans le fichier `mapping.json`.

Pour insérer un mapping, une simple requête de type `PUT` avec *cURL*, auquel on indique le chemin vers le fichier de mapping `mapping.json` en utilisant l' @ de l'option `-d` :

```
curl -XPUT http://elasticsearch:9200/map -d @mapping.json
```

## L'indexation

Ok, le mapping est rentré dans Elasticsearch, ce qui signifie que notre index `map` et notre type `shape` sont prêt à recevoir leurs premières geoshapes.
Et maintenant, grande question: Sous quel format donner les geoshapes à Elasticsearch ?

Elasticsearch se soucie de ses utilisateurs, et ne veut pas vous voir désorienté. Aussi, la geoshape se définit grâce à un format de donné appelé le geoJSON. Vous pouvez retrouver les spécifications de ce format sur le site de geoJSON.

En effet, la geoshape vous permet de stocker plusieurs sortes de figures: du simple point au polygone complexe en passant par les lignes, les cercles, et autres formes farfelues de votre imagination.

Bien sûr, je ne vais pas toutes vous les présenter, car ça n'aurait pas de sens. J'en ai choisi trois, que je vais vous introduire: La ligne (ou `LineString`), le cercle (ou `circle`) et le polygone (ou `polygon`). Chacune de ces trois geoshapes se définit par des critères bien particuliers, que nous allons voir ensemble.

### La ligne, LineString

La plus simple des trois geoshapes que je veux vous présenter est la `LineString`, ou plus simplement, une ligne.

Une ligne est définie par minimum deux points. Dans ce cas minimum, nous obtiendront une ligne simple, droite. Mais l'on peut spécifier plus de deux points afin d'obtenir une ligne brisée, pour représenter, par exemple, le plan de vol d'un avion.

Par exemple, disons que nous voulons indexer une ligne brisée qui part de Paris, qui passe par Moscou, qui redescend sur Séoul, pour terminer sur Tokyo (ce qui est approximativement le plan de vol d'un Lyon-Tokyo). Les données geoJSON de cette `LineString` sont les suivantes :

```
{
  "polygon" : {
    "type" : "linestring",
    "coordinates" : [
                      [48.995988, 2.589855],
                      [55.749724, 37.619419],
                      [37.555737, 126.988735],
                      [35.687418, 139.76326]
                    ]
  }
}
```

J'ai donc défini deux paramètres pour le champ `polygon` de notre type shape: Le premier, type défini le type de *geoshape* que nous voulons indexer (ici une `linestring`). Le deuxième, que vous retrouvez dans beaucoup de geoshapes, est le paramètres `coordinates`, qui est un tableau de coordonées. Ici, de haut en bas : Paris, Moscou, Séoul et Tokyo.
Ainsi, vous pouvez trouver ces données dans le fichier `lineString.json` à la racine du repository, et il ne nous reste plus qu'à les indexer via une query `POST`, toujours avec cURL :

```
curl -XPOST http://elasticsearch:9200/map/type/ -d @lineString.json
```

Même si pour l'instant, vous ne savez pas comment visualiser les résultats, voici ce que nous venons d'indexer:

![La linestring](/images/geoshape-elasticsearch-1/linestring.png)

### Le cercle, circle

Le cercle est l'une des geoshapes qui ne se défini pas par une tableau de point (ou alors, je vous souhaite bon courage). En revanche, un cercle en mathématiques, se défini par un point qui est son centre, et un rayon. Et c'est globalement ce que demande Elasticsearch pour définir un cercle.
Si nous voulons définir un cercle de rayon 5km autour de la Tour Eiffel, voici la requête que nous devons effectuer :

```
{
  "polygon" : {
    "type" : "circle",
    "coordinates" : [48.858136,2.294394],
    "radius" : "5km"
  }
}
```

De manière très simple, vous retrouvez le champ `type` définissant que nous voulons un `circle`, le champ `coordinates` contenant les coordonnées du centre de notre cercle, ici la Tour Eiffel, et enfin le champ `radius` qui définit un rayon de 5km.

`Ces données sont stockées dans le fichier `circle.json, toujours à la racine du repository, et on peut donc exécuter la même requête de type `POST` pour indexer notre cercle :

```
curl -XPOST http://elasticsearch:9200/map/shape -d @cirlce.json
```

Visuellement, le cercle que nous venons d'indexer ressemble à ça :

![Le circle](/images/geoshape-elasticsearch-1/circle.png)

### Le polygone, polygon

Le but de cet article, plus que d'indexer des lignes ou des cercles, est quand même de travailler avec des objets plus complexes, tels le polygone.

Dans Elasticsearch, un polygone est défini en deux temps. Tout d'abord, on fournit un tableau de coordonnées qui définissent le contour extérieur du polygone (*outer ring*). Le premier et le dernier point de ce tableau doivent être les mêmes (avoir les même coordonnées géographiques). Dans un deuxième temps, on défini un ou plusieurs tableaux de coordonnées qui représentent les "trous" dans le polygone (*inner ring*). Comme l'on est sur une sphère, bien que mise à plat, des ambiguïtés peuvent apparaître, plus particulièrement au niveau des pôles. Un organisme, l'OGC (*Open Geospatial Consortium*) a donc défini une norme :

- Le *outer ring* est tracé dans le sens contraire des aiguilles d'une montre
- Les *inner rings* sont tracés dans le sens des aiguilles d'une montre.

Vous devez donc faire attention, si vous dépassez la ligne de changement de date, alors spécifiez bien l'*outer ring* dans le sens contraire des aiguilles d'une montre, et les *inner rings* dans le sens des aiguilles d'une montre.

Si Elasticsearch détecte une ambiguïté dans la manière dont sont définis les points de votre polygone, alors il appliquera les normes de l'OGC... Ce qui peut se traduire par un polygone qui ne correspond pas à vos attentes.

Pour l'exemple, j'ai décidé d'indexer un hexagone, qui sera la France. Dans ce même polygone, je créerai un trou en définissant un *inner ring*. Pour ce faire, je vais utiliser deux tableaux de coordonnées :

Le premier est mon *outer ring*, l'hexagone qui enveloppe la France. Il part de la pointe nord de la France, et, dans le sens inverse des aiguilles d'une montre, défini chaque point de l'envelopper, pour finir par le même point que le point de départ :

```
[
     [51.069017, 2.362061],
     [48.414619, -4.779053],
     [43.34116, -1.680908],
     [42.472097, 3.109131],
     [43.802819, 7.261963],
     [48.879167, 8.162842],
     [51.069017, 2.362061]
]
```

Le deuxième tableau est un trou au milieu de la France, de forme triangulaire, défini dans les sens des aiguilles d'une montre :

```
[
     [48.618385, 2.406006],
     [45.721522, 4.844971],
     [45.73686, 3.087158],
     [48.618385, 2.406006]
]
```

Il ne reste donc qu'à écrire le document en JSON :

```
{
  "polygon" : {
    "type" : "polygon",
    "coordinates" : [
                      [
                        [51.069017, 2.362061],
                        [48.414619, -4.779053],
                        [43.34116, -1.680908],
                        [42.472097, 3.109131],
                        [43.802819, 7.261963],
                        [48.879167, 8.162842],
                        [51.069017, 2.362061]
                      ],
                      [
                        [48.618385, 2.406006],
                        [45.721522, 4.844971],
                        [45.73686, 3.087158],
                        [48.618385, 2.406006]
                      ]
                    ]
  }
}
```

Vous trouverez ce document JSON dans le fichier `polygon.json` à la racine de repository. Une petite requête cURL pour indexer le document, et le tour est joué :

```
curl -XPOST http://elasticsearch:9200/map/shape -d @polygon.json
```

La geoshape polygonale que nous venons d'indexer ressemble à ça :

![Le polygon](/images/geoshape-elasticsearch-1/polygon.png)

Le triangle vert représente le "trou" que nous avons créé dans le polygone, en définissant un outer ring. L'index contient donc maintenant ce polygone, et un simple petit tour avec Head sur votre Elasticsearch vous le confirmera.

# Et ensuite ?

Les geoshapes d'Elasticsearch sont un sujet assez vaste. Je vous ai présenté 3 des geoshapes disponibles, et la manière d'indexer une geoshape, qui reste sensiblement la même avec toutes les geoshapes.
Dans la deuxième partie, je ne reviendrai pas sur l'indexation, mais je parlerai des requêtes possibles avec Elasticsearch et les geoshapes (ce qui promet d'être fun !)

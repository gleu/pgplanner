Bitmap Scan
===========

One of the biggest issue of the Index Scan is the disk head movement to scan
the index, and to scan the table. Each interesting record on the index will
have to be checked for visibility in the table. This is very bad for
performance.

PostgreSQL developers tried to lower the performance impact by scanning the
index first, and storing temporarily the record pointers in memory. Once the
index is read, it is easy and quick to scan the table, sequentially, according
to the record pointer in memory, in physical order of the table, and not in
logical order of the records in the index.

This scan type is called a Bitmap Index Map because it creates an bitmap index
in memory. This is a field of bits: value 0 means the record doesn't
match the filter, value 1 means it matches. If there are too many records to
store, the bitmap becomes lossy, and each stored bit represents a page, rather
than a tuple. In such a case, during the table scan, a recheck has to be done
on the tuple level (the executor knows some of the rows in this page match the
filter, but it needs to find which one does). The advantage is that the
executor can work on a huge number of rows without using too much memory.

Comme la construction de ce bitmap se fait en parcourant un index, les informations
n'arrivent pas dans l'ordre physique des lignes et blocs de la table mais dans l'ordre
physique de l'index, ce qui n'est pas idéal pour créer un bitmap trié tel qu'on le
voudrait. Afin de créer cette structure de manière efficace, la construction se fait en
deux temps. Lors du parcours initial de l'index, la partie du bitmap correspondant
au bloc de la ligne à mémoriser est d'abord stockée dans une table de hachage.
Cela permet de retrouver cette même partie très rapidement pour les prochaines
lignes correspondant au même bloc. Une fois que le parcours initial de l'index est
effectué, la table de hachage est parcourue séquentiellement par ordre croissant
de sa clé, c'est-à-dire par ordre croissant des blocs de la table correspondante. Il
suffit donc de chaîner les différentes parties du bitmap pour générer le bitmap final
prêt à être utilisé.

La table de hachage utilisée lors de la création d'un bitmap a quelques particulari-
tés par rapport à d'autres tables de hachage utilisées par PostgreSQL. Tout d'abord,
elle aura toujours la même structure. Ainsi, la clé de la table de hachage est toujours
la même, de même que la fonction permettant de comparer cette clé. De plus, il n'est
pas rare de se retrouver à stocker un très grand nombre d'entrées. Or, PostgreSQL
dispose d'une implémentation généraliste de table de hachage, appelée dynahash ,
permettant de gérer beaucoup de cas possibles. Mais celle-ci n'est pas très perfor-
mante dans ce cas particulier. C'est pourquoi un autre type de table de hachage
a été introduit avec la version 10 de PostgreSQL. Cette nouvelle implémentation,
appelée simplehash , permet de définir à la compilation et non à l'exécution le type
de la clé ainsi que les différentes fonctions nécessaires pour utiliser cette table de
hachage, ce qui évite un surcoût inutile. De plus, la gestion des collisions est pensée
pour être efficace dans le cas de très grosses tables de hachage, où les collisions
sont fréquentes. Tout cela contribue à grandement améliorer les performances de
ce nœud en version 10.

Note > Contrairement à d'autres moteurs de base de données, PostgreSQL ne permet pas de stocker
l'index bitmap sur disque. Ce sont uniquement des représentations en mémoire des index existants.

Ce type de parcours est composé d'au moins deux nœuds. Le premier, un Bit-
map Index Scan fait le parcours de l'index et crée l'index bitmap en mémoire. Le
deuxième, appelé Bitmap Heap Scan , lit les blocs pointés par l'index bitmap dans
la table pour vérifier le respect du prédicat des lignes de ce bloc. Ainsi, il récupère
les lignes à renvoyer. En voici un exemple simple :

Bitmap Heap Scan on t1 ...
Recheck Cond: (c1 = ANY ('{10,40,60,100,600}'::integer[]))
-> Bitmap Index Scan on t1_c1_idx ...
Index Cond: (c1 = ANY ('{10,40,60,100,600}'::integer[]))

Le parcours de table étant séquentiel et partiel (uniquement pour les blocs intéres-
sants), il est bien plus rapide qu'un parcours d'index comme le Index Scan pour un
nombre moyen de lignes.

Donc son premier intérêt est purement la performance du parcours due au décou-
plage des parcours de l'index et de la table. Il présente néanmoins un autre intérêt.

Soit une requête utilisant un filtre sur deux colonnes :

SELECT * FROM t1 WHERE c1<100000 AND c2>=50

Une autre force de ce type de parcours est qu'il est très facile de cumuler plusieurs
de ces bitmaps pour récupérer les lignes voulues. En effet, s'il existe un index sur
c1 et un index sur c2 , PostgreSQL ne pourra pas faire un parcours d'index sur ces
deux index à la fois. Cependant, combiner deux champs de bits en mémoire est
très efficace. Il s'agit d'opérations AND ou OR booléennes très simples et très rapides.
Dans ce cas, l'exécuteur calcule un index bitmap pour l'index de c1 et un index
bitmap pour l'index de c2 . Ceci fait, il ne lui reste plus qu'à faire un AND logique
(nœud BitmapAnd ) sur les deux champs de bits s'il faut retourner les lignes corres-
pondant aux deux prédicats, ou un OR logique (nœud BitmapOr ) pour les lignes
correspondant à au moins un prédicat, et il se retrouve avec un unique champ de
bits qu'il sait traiter avec un nœud Bitmap Heap Scan .

Bitmap Heap Scan on clients
...
Recheck Cond: ((c1 < 100000) AND (c2 > 50))
->
BitmapAnd
...
-> Bitmap Index Scan on t1_c1_idx ...
-> Index Cond: (c1 < 100000)
Bitmap Index Scan on t1_c2_idx
Index Cond: (c2 > 50) ...
est le nom du nœud parent. Il s'occupe de renvoyer les
données vérifiées auprès de la table.

La ligne Recheck Cond indique que le filtre complet est vérifié au niveau de la
table. Cette opération est nécessaire lorsque le nombre de lignes est trop im-
portant et qu'on est passé d'un index bitmap d'enregistrements à un index bit-
map de blocs. Ce changement de type de bitmap est très préjudiciable pour
les performances. Une augmentation de la valeur du paramètre work_mem est
le seul moyen de l'éviter.

La ligne BitmapAnd indique qu'une opération AND booléenne est exécutée
entre deux index bitmaps. Les deux parcours d'index bitmaps sont les enfants
de ce nœud.

Bitmap Heap Scan

Il est à noter qu'il existe aussi le nœud BitmapOr quand le lien entre deux
filtres est un OR booléen.

Ces deux lignes sont les enfants directs de la ligne BitmapAnd . Ce sont des
parcours d'index qui sont représentés en mémoire par un champ de bits. Cha-
cun des bits correspond à une ligne (ou à un bloc si le nombre de lignes est
trop important).

La ligne Index Cond indique le filtre utilisé. Dans le cas de cette ligne, Post-
greSQL utilise l'index pour trouver rapidement les lignes satisfaisant le filtre
c1 < 1000 .

Du fait d'avoir à d'abord créer l'index bitmap en mémoire, la récupération des
premières lignes est lentes (on dit que le coût de démarrage est élevé), ce parcours
est moins intéressant dans le cas de l'utilisation de curseurs ou de la clause LIMIT .
Son efficacité est augmentée si les disques utilisés peuvent gérer un grand nombre
d'opérations d'entrées/sorties disque en parallèle. Dans ce cas, il faut configurer
le paramètre effective_io_concurrency avec une valeur supérieure à zéro. Pour
des disques SSD, la valeur peut atteindre plusieurs centaines.

Le paramètre enable_bitmapscan permet de désactiver temporairement les par-
cours d'index bitmap. Il est essentiel de ne pas les désactiver globalement en pro-
duction.

Voici une explication “manager-friendly” puis technique, pour étayer pourquoi Cobrix (a.k.a. spark-cobol) développé et publié pour Spark 3 peut poser des problèmes si on bascule tel quel vers Spark 4.

Résumé pour vos chefs (TL;DR)

Cobrix 2.6+ cible Spark 3 (≥ 3.2.0). C’est l’environnement explicitement supporté par le projet aujourd’hui. Rien n’indique un support officiel Spark 4 dans la doc actuelle. 
GitHub

Spark 4 change le socle technique (Scala 2.13 uniquement, Java 17, dépendances Parquet/Jackson, etc.) et active par défaut l’ANSI SQL. Ces ruptures de compatibilité peuvent casser des librairies compilées pour Spark 3 et/ou changer le comportement de nos jobs. 
spark.apache.org
+1

Conséquence: risque élevé d’incompatibilités binaires et fonctionnelles (échec de chargement des JAR, conflits de dépendances, erreurs à l’exécution). On recommande de rester en Spark 3 pour les workloads Cobrix, ou de prévoir un POC de requalification (rebuild Cobrix pour Scala 2.13 / Java 17, tests de régression, etc.) avant toute montée majeure. 
spark.apache.org
GitHub

Détails techniques (pour experts Spark)
1) Matrice de compatibilité actuelle du projet Cobrix

Le README officiel liste les prérequis suivants :

Cobrix 2.6.x+ ⇒ Spark ≥ 3.2.0 (la ligne “2.6.x+ → 3.2.0+”).

Artefacts publiés pour Scala 2.11 / 2.12 / 2.13 (ex. 2.8.4).
=> Cobrix est conçu/testé avec Spark 3 ; aucun engagement explicite sur Spark 4 aujourd’hui. 
GitHub

2) Changements majeurs côté Spark 4 qui cassent la compatibilité

Fin de Scala 2.12 : Scala 2.13 par défaut → tous les JARs compilés contre Spark 3 + Scala 2.12 doivent être repackagés pour 2.13 pour éviter NoSuchMethodError / ClassNotFoundException. 
spark.apache.org

Fin de JDK 8/11 : Java 17 par défaut → impacte la toolchain (compilation, CI/CD, runtime sur le cluster). 
spark.apache.org

Montée de versions transitive massive (Parquet, Jackson, Py4J, RocksDB, etc.) → risque de conflits avec les versions embarquées par des dépendances de Cobrix ou de notre plateforme. 
spark.apache.org

ANSI SQL activé par défaut → comportements différents (ex. dépassements numériques, division par zéro, conversions implicites) qui pouvaient “passer” en Spark 3 et échouent en Spark 4. Même si Cobrix lit le binaire COBOL, les DataFrames résultants et nos requêtes peuvent être affectés. 
spark.apache.org

Évolution du framework DataSource V2 : Spark 4 pousse encore V2, les ponts V1→V2 restent mais sont instables à terme. Or, l’implémentation Cobrix est historiquement un DataSource externe “format cobol” (au minimum V1). Un changement ou une suppression future du pont peut exiger une adaptation du code. 
spark.apache.org
+1

3) Risques concrets qu’on observe typiquement en montée majeure

Incompatibilité binaire au chargement : NoSuchMethodError / ClassNotFoundException lors du spark-submit parce que Cobrix a été compilé contre des signatures Spark 3 / Scala 2.12. 
spark.apache.org

Conflits de dépendances (“jar hell”) : versions Parquet/Jackson différentes entre Spark 4 et ce que Cobrix (ou nos libs) attendent → erreurs au runtime ou corruption silencieuse. 
spark.apache.org

Changements de sémantique SQL : jobs existants qui passaient sous Spark 3 échouent sous Spark 4 (ANSI). 
spark.apache.org

Évolution DataSource : si Cobrix s’appuie sur des API internes ou des comportements V1, réécriture partielle possible pour tirer parti de V2 et garantir la pérennité. 
spark.apache.org
+1

4) Ce qu’on peut montrer comme “preuves”

Doc Cobrix : prérequis affichés (Cobrix 2.6.x+ → Spark 3.2.0+), artefacts 2.8.4 pour Scala 2.11/2.12/2.13. 
GitHub

Release notes Spark 4.0 :

Drop Scala 2.12 (Scala 2.13 par défaut)

Drop JDK 8/11 (Java 17 par défaut)

ANSI SQL mode par défaut

Mises à jour de dépendances clés
Ces points ≈ ruptures vs Spark 3. 
spark.apache.org

Recommandation pragmatique

Garder Spark 3 pour les flux Cobrix en prod (idéalement un Spark 3.5.x LTS) le temps de valider.

Monter un POC ciblé Spark 4 :

Recompiler/packager Cobrix pour Scala 2.13 et Java 17 (artefact _2.13).

Exécuter notre jeu de tests fonctionnels sur un cluster Spark 4.

Vérifier les points sensibles : parsing Cobol multi-segments, encodages EBCDIC/ASCII, RDW, streaming, et nos requêtes SQL avec ANSI activé.

Écarter les conflits de dépendances (Parquet/Jackson) via --packages / --jars contrôlés et un BOM Maven/Gradle propre. 
Maven Repository
spark.apache.org

Plan B si blocage :

Isoler les jobs Cobrix sur un pool Spark 3 jusqu’à annonce officielle de compatibilité Spark 4 par les mainteneurs.

Ou fork et adapter le DataSource vers V2 pour durcir la compatibilité long terme.

Formulation “claire” à communiquer

« Cobrix a été développé et validé pour Spark 3 (≥ 3.2). Spark 4 introduit des changements de base (Scala 2.13, Java 17, dépendances mises à jour, ANSI SQL par défaut) qui ne garantissent pas la compatibilité binaire/fonctionnelle de nos jobs. Sans requalification, on risque des erreurs de chargement et des changements de comportement. Nous recommandons de conserver Spark 3 pour les flux Cobrix en production et de mener un POC de revalidation technique avant toute migration. »

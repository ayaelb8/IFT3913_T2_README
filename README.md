# Tâche 2

## Classe à tester:

Le choix de classes devait être basé sur la présence de tests pour la classe en question et une couverture de test inférieure à 100%. La classe __`MediaType.java`__, présente dans le module tika-core, a été choisie entre autres puisque son fichier de test existe[^1]. De plus, l'index[^2] de couverture avec JaCoCo indique une __couverture d'instructions de 79%__ et une couverture de branches de 74%. De plus, le _Pit Test Coverage Report_[^3] nous indique une __couverture de mutation de 54 %__, soit 46/85. En effet, malgré une couverture d'instruction de 79 %, 46 mutants sont tués, __6 mutants survivent__ et 33 mutants (38,82 %) n'ont pas pu être détectés par manque de tests adéquats exécutant le code correspondant à chaque mutant. Les mutants survivants montrent que certains tests ont réussi malgré la modification introduite par PIT, ce qui indique ainsi des comportements qui ne sont pas suffisamment vérifiés par les tests. Cela nous permet d'avoir des pistes sur les améliorations ou les ajouts à faire pour améliorer les tests existants. 

La classe `MediaType.java` comporte 28 méthodes dont 11 avec une couverture inférieure à 100%.
<img width="989" height="659" alt="image" src="https://github.com/user-attachments/assets/f44fa766-0859-4e8b-a8ba-f2b532704aef" />[^4]

La méthode `isSimpleName()` est la seule à avoir des mutants survivants dans `MediaType.java`, cette méthode a une couverture de 97%. Cette méthode est privée, donc les tests seront générés pour la méthode publique `MediaType.parse()` qui appelle `isSimpleName()` aux lignes 259 à 260.
<img width="1014" height="220" alt="image" src="https://github.com/user-attachments/assets/2d23fbb8-0e69-4282-8a2d-6ce72da7a7c6" />
<img width="1251" height="839" alt="image" src="https://github.com/user-attachments/assets/761c7dfc-d9c5-4f4b-8253-45202ef4b40d" />[^5]
```
public static MediaType parse(String string) {
        if (string == null) {
            return null;
        }

        // Optimization for the common cases
        synchronized (SIMPLE_TYPES) {
            MediaType type = SIMPLE_TYPES.get(string);
            if (type == null) {
                int slash = string.indexOf('/');
                if (slash == -1) {
                    return null;
                } else if (SIMPLE_TYPES.size() < 10000 &&
                        isSimpleName(string.substring(0, slash)) &&
                        isSimpleName(string.substring(slash + 1))) {
                    type = new MediaType(string, slash);
                    SIMPLE_TYPES.put(string, type);
                }
            }
            if (type != null) {
                return type;
            }
        }

        Matcher matcher;
        matcher = TYPE_PATTERN.matcher(string);
        if (matcher.matches()) {
            return new MediaType(matcher.group(1), matcher.group(2),
                    parseParameters(matcher.group(3)));
        }
        matcher = CHARSET_FIRST_PATTERN.matcher(string);
        if (matcher.matches()) {
            return new MediaType(matcher.group(2), matcher.group(3),
                    parseParameters(matcher.group(1)));
        }

        return null;
    }
```
[^6]

__Pour la génération de test, la méthode choisie sera donc logiquement indirectement la méthode `isSimpleName()` via `MediaType.parse()` dans la classe `MediaType.java` et le module tika-core.__

## IA et test

### Installation de ChatUniTest dans le pipeline Maven

ChatUniTest est un plugin Maven qui génère des tests JUnit à l'aide d'un modèle de langage (LLM). Nous l'avons intégré au module `tika-core`, dans le même `pom.xml` que PIT, pour générer des tests pour la classe `org.apache.tika.mime.MediaType`.

Deux éléments ont été ajoutés à `tika-core/pom.xml`, comme le demande la documentation du plugin :

1. __La dépendance `chatunitest-starter` 1.4.0__ (`io.github.ZJU-ACES-ISE`, type `pom`), qui fournit les bibliothèques utilisées par les tests générés (JUnit, Mockito). Nous l'avons placée en portée `test` pour qu'elle ne soit pas transmise aux projets qui dépendent de `tika-core`.
2. __Le plugin `chatunitest-maven-plugin` 2.1.1__ (`io.github.zju-aces-ise`), la dernière version publiée sur Maven Central, configuré ainsi :

| Paramètre | Valeur | Rôle |
|---|---|---|
| `apiKeys` | `ollama` | Obligatoire pour le plugin; un modèle local ne la vérifie pas |
| `model` | `codeqwen:v1.5-chat` | Nom de modèle accepté par le plugin (voir défi 3) |
| `url` | `http://localhost:11434/v1/chat/completions` | API compatible OpenAI du serveur Ollama local |
| `selectClass` | `org.apache.tika.mime.MediaType` | Classe pour laquelle générer des tests |
| `thread` | `false` | Requêtes envoyées une à la fois (voir défi 4) |
| `maxPromptTokens` | `3000` | Taille maximale des demandes (voir défi 5) |
| `testNumber` | `5` | Nombre maximal de tests tentés par méthode |
| `maxRounds` | `5` | Nombre maximal de corrections demandées au LLM par test |
| `temperature` | `0.5` | Valeur par défaut |

Le LLM est un modèle local servi par Ollama, `qwen2.5-coder:3b`, plutôt qu'une API payante.

La génération se lance depuis la racine du projet, avec le Maven wrapper fourni par Tika et un JDK 17 :

```
.\mvnw.cmd -pl tika-core -am install -DskipTests
.\mvnw.cmd -pl tika-core chatunitest:class
```

La première commande compile `tika-core` et les modules dont il dépend. La seconde analyse le module (361 classes, 1617 méthodes), puis, pour chaque méthode de `MediaType`, demande un test au LLM, le compile et l'exécute. En cas d'échec, l'erreur est renvoyée au LLM pour correction, jusqu'à 5 fois.

### Défis rencontrés lors de l'installation

1. __Environnement Java et Maven.__ Maven n'était pas installé et `JAVA_HOME` n'était pas défini. Nous avons utilisé le Maven wrapper du projet (`mvnw.cmd`). L'option `-am` est nécessaire : sans elle, le build échoue parce que `tika-core` dépend du module `tika-annotation-processor`.
2. __Incompatibilité avec Java 27.__ Le plugin échouait avec `Invalid Java version format: 27` : il lit la version de Java en la découpant sur le caractère « . » et rejette une version sans point. Nous avons installé le JDK 17 (Temurin 17.0.20), qui est aussi la version ciblée par Tika.
3. __Liste fermée de noms de modèles.__ ChatUniTest 2.1.1 n'accepte que quelques noms de modèles prédéfinis (`gpt-4o-mini`, `code-llama`, `codeqwen:v1.5-chat`, etc.) et refuse `qwen2.5-coder`. Nous avons créé dans Ollama un alias portant un nom accepté : `ollama cp qwen2.5-coder:3b codeqwen:v1.5-chat`.
4. __Délais d'attente dépassés.__ Par défaut, le plugin envoie une dizaine de requêtes en parallèle, alors que le modèle local les traite une à la fois. Les requêtes en attente dépassaient le délai HTTP du plugin, fixé à 5 minutes dans son code et non configurable (`SocketTimeoutException`). Nous avons désactivé le multithreading (`thread=false`).
5. __Demandes tronquées.__ Ollama utilisait une fenêtre de contexte de 4096 tokens, alors que le plugin pouvait envoyer des demandes allant jusqu'à 10 923 tokens. Ollama tronque l'excédent sans signaler d'erreur. Nous avons limité les demandes à 3000 tokens (`maxPromptTokens`), ce qui laisse la place aux 1024 tokens de réponse.
6. __Limites matérielles.__ C'est le défi qui a le plus pesé sur le résultat, parce qu'il a dicté à la fois le choix du modèle et la quantité de code que le modèle pouvait lire.
   - __La machine.__ La génération a été faite sur un ordinateur portable : processeur Intel Core i5-9300H (4 cœurs), 16 Go de mémoire vive et carte graphique NVIDIA GTX 1650 avec 4 Go de mémoire vidéo. Nous n'avions pas accès à une API payante ni à une machine plus puissante.
   - __Le premier modèle ne tenait pas dans la carte graphique.__ `qwen2.5-coder:7b` occupe environ 5 Go une fois chargé, soit plus que les 4 Go disponibles. Ollama le répartissait à 55 % sur le processeur et 45 % sur la carte graphique. Les réponses étaient alors très lentes, et c'est ce qui provoquait les délais d'attente dépassés du défi 4, fixés à 5 minutes. Selon notre estimation, une génération complète aurait pris plusieurs heures.
   - __Le modèle retenu est plus petit.__ Nous sommes passés à `qwen2.5-coder:3b` (environ 2 Go chargé), qui s'exécute entièrement sur la carte graphique. La génération complète devient possible en un temps raisonnable, mais avec un modèle qui a moins de la moitié des paramètres du premier, donc moins capable de raisonner sur du code.
   - __Le modèle lit peu de code à la fois.__ Pour tenir dans la mémoire vidéo, la fenêtre de contexte est restée à 4096 tokens. C'est ce qui nous a obligés à limiter les demandes à 3000 tokens (défi 5), alors que ChatUniTest est prévu pour en envoyer jusqu'à 10 923. Le plugin envoie donc au modèle moins d'informations sur `MediaType` et sur les classes qu'elle utilise.

   Ces contraintes expliquent vraisemblablement une bonne partie des défauts relevés dans la section « Explication et critique des tests générés ». Un petit modèle qui ne voit qu'une partie du code a tendance à compléter avec ce qu'il suppose : il invente une validation qui n'existe pas, se trompe sur le rôle des arguments, et ajoute Mockito par habitude. Nous n'avons toutefois pas pu le vérifier : il aurait fallu refaire la génération avec un modèle plus gros dans la même configuration, ce que la machine ne permettait pas. Les résultats de cette tâche décrivent donc ce que ChatUniTest produit avec un modèle local de 3 milliards de paramètres, et non ce que l'outil peut produire avec un modèle plus puissant.

### Résultat de la génération

__Où sont les tests générés ?__ ChatUniTest a écrit 9 fichiers dans `tika-core/chatunitest-tests/org/apache/tika/mime/` : 8 classes de test, nommées `MediaType_<méthode>_<n>_<n>_Test.java`, et `MediaType_Suite.java`, qui les regroupe. Elles visent 8 des 17 méthodes publiques de `MediaType` : `application`, `audio`, `image`, `text`, `parse`, `equals`, `compareTo` et `hasParameters`. Aucune classe n'a été produite pour `video`, `set` (deux variantes), `getType`, `getSubtype`, `getBaseType`, `getParameters`, `hashCode` et `toString`. Ce dossier ne fait pas partie des sources de test de Maven : pour exécuter les tests, il faut les copier dans `tika-core/src/test/java/org/apache/tika/mime/`.

__Les tests générés compilent-ils et s'exécutent-ils sans intervention manuelle ?__ Non. Les 8 classes de test compilent telles quelles, mais pas le fichier de suite, et l'exécution a demandé des corrections de la configuration Maven. Les problèmes et les corrections sont décrits dans les deux sections suivantes. Une fois ces corrections faites, les 95 tests s'exécutent, mais seulement 28 réussissent (48 échecs et 19 erreurs). Le détail est donné dans la section « Résultats et analyse des tests générés ».

### Problème à la suite de la génération de tests

En intégrant les tests générés par ChatUniTest, j’ai également obtenu le fichier `MediaType_Suite.java`, qui regroupe les différentes classes de tests. Cependant, ce fichier utilise des imports JUnit Platform (`org.junit.platform.runner` et `org.junit.platform.suite.api`) qui ne sont pas disponibles dans la configuration actuelle du projet. Cela entraîne une erreur de compilation. Pour pouvoir évaluer les huit classes de tests individuellement sans modifier le contenu généré, j’ai déplacé le fichier `MediaType_Suite.java` hors du répertoire des tests pendant l'exécution des autres fichiers. Ce problème d’intégration a été noté séparément des échecs et des erreurs observés lors de l’exécution des tests eux-mêmes.

Dans `MediaType_audio_1_2_Test`, un test a également provoqué une erreur liée à un nombre incorrect d'arguments lors d'un appel par réflexion. Il s'agit là encore d'un problème dans la manière dont le test invoque la méthode.

### Corrections nécessaires

__Combien de corrections ont été nécessaires ?__ Quatre, toutes dans la configuration ou l'intégration. Le contenu des 8 classes générées n'a pas été modifié.

| # | Problème observé | Correction |
|---|---|---|
| 1 | Avec `chatunitest-starter`, `mvn test` n'exécutait plus aucun test de `tika-core` (`Tests run: 0`), y compris `MediaTypeTest`, tout en affichant `BUILD SUCCESS`. Le starter apporte JUnit 4 et `junit-platform-runner`, ce qui fait choisir à Surefire son fournisseur JUnit 4, qui ignore les tests JUnit 5. | Exclusion de `junit-platform-runner` de la dépendance `chatunitest-starter`. |
| 2 | Même cause que le problème 1. | Configuration explicite de `maven-surefire-plugin` 3.5.6 pour le moteur `junit-jupiter` (`includeJUnit5Engines`). |
| 3 | Les tests générés qui utilisent Mockito échouaient tous à la création des mocks : 31 erreurs sur 95 tests, soit tous les tests de `audio`, `hasParameters` et `parse`. | Exclusion de la version de Byte Buddy apportée par le starter, et ajout de `byte-buddy` et `byte-buddy-agent` 1.17.7 en portée `test`. Les erreurs passent de 31 à 19. |
| 4 | Le fichier de suite généré ne compile pas (problème décrit dans la section précédente). | Mise à l'écart du fichier de suite; les 8 classes sont exécutées individuellement. |

Après ces corrections, les tests d'origine s'exécutent de nouveau normalement (`MediaTypeTest` : 9 tests, 9 réussis).

Les 67 tests générés qui échouent encore n'ont pas été corrigés : leurs défauts viennent du contenu des tests eux-mêmes (assertions qui ne correspondent pas au comportement réel de `MediaType`, usage inutile de Mockito) et sont analysés plus bas. Par exemple, les 16 erreurs de `MediaType_parse_7_0_Test` viennent toutes de l'annotation `@InjectMocks` appliquée à `MediaType`, que Mockito ne peut pas instancier.

## Documentation des tests générés

### Résultats et analyse des tests générés

| Classe de test | Tests exécutés | Échecs | Erreurs |
|---|---:|---:|---:|
| `MediaType_application_0_0` | 28 | 26 | 0 |
| `MediaType_audio_1_2` | 5 | 3 | 1 |
| `MediaType_compareTo_20_1` | 5 | 3 | 0 |
| `MediaType_equals_18_1` | 11 | 0 | 0 |
| `MediaType_hasParameters_15_3` | 10 | 8 | 2 |
| `MediaType_image_2_1` | 9 | 8 | 0 |
| `MediaType_parse_7_0` | 16 | 0 | 16 |
| `MediaType_text_3_1` | 11 | 0 | 0 |
| **Total** | **95** | **48** | **19** |

Les huit classes de tests générées par ChatUniTest ont été exécutées individuellement, sans modifier leur contenu. D'après les rapports[^7], sur les 95 tests exécutés, 28 ont réussi, 48 ont échoué et 19 ont provoqué des erreurs d’exécution. Un test qui réussit n’est pas automatiquement bon, et inversement, un test qui échoue n’est pas automatiquement mauvais. Il faut vérifier les assertions et les comparer au comportement réel de MediaType. Les classes __`MediaType_equals_18_1_Test`__ et __`MediaType_text_3_1_Test`__ ont réussi tous leurs tests. 

Dans __`MediaType_application_0_0_Test`__, 26 des 28 tests ont échoué. Par exemple, `testNullApplication()` attendait null, alors que le résultat obtenu était application/null. D’autres tests, comme `testApplicationWithCharset()` et `testApplicationWithParameters()`, attendaient une valeur non nulle, mais la méthode retournait null pour les entrées utilisées. Ces résultats montrent que les cas de test et les résultats attendus doivent être vérifiés à partir du comportement réel de la méthode `application()`.

Dans __`MediaType_compareTo_20_1_Test`__, trois des cinq tests ont échoué. Le test `testCompareToDifferentTypeSameParameters()` attendait exactement -1, alors que la méthode retournait 19. Or, la méthode `compareTo()` doit notamment respecter le signe correspondant à l'ordre de comparaison; elle ne garantit pas nécessairement une valeur exacte de -1 ou de 1. Deux autres tests attendaient une NullPointerException, mais aucune exception n'a été lancée. Ces assertions reposent donc sur des hypothèses qui ne correspondent pas au comportement observé.

Dans __`MediaType_image_2_1_Test`__, huit des neuf tests ont échoué. Par exemple, `testImageMethodWithInvalidType()` attendait null, alors que le résultat était image/invalid. De même, `testImageMethodWithParameters()` attendait charset=utf-8, alors que la valeur récupérée était utf-8. Ces exemples indiquent que certains tests supposent une validation plus stricte que celle effectuée par la méthode ou comparent incorrectement la valeur d'un paramètre.

Enfin, dans __`MediaType_hasParameters_15_3_Test`__, huit tests ont échoué et deux ont provoqué des erreurs. Plusieurs tests attendaient true pour des objets comportant des paramètres, mais obtenaient false. Deux autres tests ont échoué à cause de configurations Mockito jugées inutiles.

### Explication et critique des tests générés

Cette section explique ce que chaque classe générée cherche à vérifier, puis critique la qualité des tests à partir du code de `MediaType`[^6]. Il faut savoir que les méthodes `application()`, `audio()`, `image()` et `text()` font toutes la même chose : elles ajoutent un préfixe au sous-type reçu et appellent `parse()`. Par exemple, `image("jpeg")` retourne `parse("image/jpeg")`.

| Classe de test | Ce que la classe vérifie | Réussis | Critique principale |
|---|---|---:|---|
| `MediaType_application_0_0` | `application()` avec des types valides, avec paramètres, invalides, nuls et vides | 2 / 28 | Passe un type complet (`application/json`) au lieu d'un sous-type (`json`) |
| `MediaType_audio_1_2` | `audio()` avec un type valide, nul, vide et invalide | 1 / 5 | Appelle par réflexion une méthode publique; mock inutilisé |
| `MediaType_compareTo_20_1` | L'ordre entre des types égaux, différents et `null` | 2 / 5 | Attend la valeur exacte `-1` et des exceptions qui ne sont jamais lancées |
| `MediaType_equals_18_1` | L'égalité selon le type, le sous-type et les paramètres | 11 / 11 | Valide, mais ne teste ni `null` ni un objet d'une autre classe |
| `MediaType_hasParameters_15_3` | `hasParameters()` avec zéro, un ou plusieurs paramètres | 0 / 10 | Teste un mock de `MediaType`, et non la vraie classe |
| `MediaType_image_2_1` | `image()` avec un sous-type valide, des paramètres et des entrées invalides | 1 / 9 | Suppose une validation qui n'existe pas |
| `MediaType_parse_7_0` | `parse()` avec des types simples, des paramètres et des valeurs entre guillemets | 0 / 16 | Aucun test ne s'exécute à cause de `@InjectMocks` |
| `MediaType_text_3_1` | `text()` avec des entrées jugées invalides | 11 / 11 | Réussit par accident; 6 tests identiques; aucun cas valide |

#### Défauts observés

__1. Le modèle invente la spécification au lieu de lire le code.__ C'est la cause de la plupart des 48 échecs. Les tests supposent que `MediaType` valide ses entrées et retourne `null` pour un type inconnu, alors que le code accepte tout nom syntaxiquement correct : `image("invalid")` retourne `image/invalid`, et `application(null)` retourne `application/null`. De même, `MediaType_compareTo_20_1_Test` attend une `NullPointerException` quand les paramètres diffèrent, et `MediaType_parse_7_0_Test` attend une `IllegalArgumentException` pour `parse(null)`, alors que la méthode retourne `null`.

__2. Les méthodes sont mal utilisées.__ `MediaType_application_0_0_Test` appelle `application("application/json")`, ce qui donne `application/application/json`, que `parse()` rejette. Ses assertions sont aussi incohérentes : elles attendent que `getType()` retourne le type complet et que `getSubtype()` retourne une chaîne vide. `MediaType_equals_18_1_Test` écrit `new MediaType("text/plain", "utf-8")` en croyant fournir un type et un jeu de caractères, alors que le second argument du constructeur est le sous-type.

__3. Mockito et la réflexion sont utilisés sans raison, et cassent des tests.__ `MediaType` est une classe de valeur sans dépendance : il n'y a rien à simuler. Pourtant :
- `MediaType_hasParameters_15_3_Test` appelle `hasParameters()` sur un mock. C'est Mockito qui répond `false`, et le code de `MediaType` n'est jamais exécuté. Même réussis, ces 10 tests ne vérifieraient rien.
- `MediaType_parse_7_0_Test` annote un champ `MediaType` avec `@InjectMocks`. Mockito ne peut pas l'instancier, et les 16 tests tombent en erreur avant de s'exécuter.
- `MediaType_audio_1_2_Test` appelle `audio()` par réflexion, avec le commentaire « private audio method », alors que la méthode est publique. Un des tests lui passe deux arguments au lieu d'un, d'où l'erreur `wrong number of arguments`.

__4. Des tests réussissent sans rien démontrer.__ Les 11 tests de `MediaType_text_3_1_Test` passent tous, mais pour une mauvaise raison. Ils appellent par exemple `text("text/plain; charset=utf-8; invalid")`, ce qui donne `text/text/plain; ...` : le résultat est `null` à cause du préfixe en double, et non à cause du paramètre « invalide » que le test croit vérifier. Aucun test ne vérifie le cas normal, `text("plain")`. C'est un exemple concret du fait qu'un test qui réussit n'est pas forcément un bon test.

__5. Les tests sont redondants et mal nommés.__ Dans `MediaType_text_3_1_Test`, 6 des 11 tests utilisent exactement la même entrée sous des noms différents (`...InvalidKeyAndValue`, `...InvalidKeyAndValue2`, `...3`, `...4`). Dans `MediaType_equals_18_1_Test`, 9 des 11 tests vérifient la même propriété (deux objets différents ne sont pas égaux) avec des combinaisons très proches, et `testEqualsWithDifferentOrderOfParameters` n'échange pas l'ordre des paramètres : il inverse la clé et la valeur. Le nombre de tests donne donc une fausse impression de couverture.

__6. Les tests ne visent pas l'objectif de la tâche.__ Les mutants survivants se trouvent dans `isSimpleName()`, accessible par `parse()`. Les tuer demande des entrées aux limites des caractères acceptés (par exemple une majuscule, un espace, ou les caractères `-`, `+`, `.` et `_`). Aucun test généré ne cible ces limites de façon délibérée, et la seule classe consacrée à `parse()` ne s'exécute pas. Nous n'avons pas mesuré le score de mutation des tests générés, mais leur contenu ne permet pas d'attendre une amélioration sur ces mutants.

#### Points positifs

- Les 8 classes compilent sans modification et suivent les conventions du projet (JUnit 5, même paquet que la classe testée).
- Les tests sont lisibles : noms descriptifs, structure préparation / action / vérification, tests paramétrés avec `@CsvSource` pour varier les entrées.
- `MediaType_equals_18_1_Test` est utilisable tel quel : ses assertions correspondent au comportement réel de `equals()`.
- Le modèle pense à des catégories de cas pertinentes (valeur nulle, chaîne vide, paramètres, guillemets), même si les résultats attendus sont souvent faux.

#### Bilan

Sur 95 tests, 28 réussissent, et seule une partie d'entre eux vérifie réellement quelque chose : les 11 tests de `equals()`, et quelques tests isolés des autres classes. Les tests générés par un petit modèle local (`qwen2.5-coder:3b`) peuvent servir de point de départ pour trouver des idées de cas, mais chaque résultat attendu doit être revérifié à partir du code. Ce constat est à lire avec les limites matérielles décrites plus haut (défi 6) : la taille du modèle et le peu de code qu'il pouvait lire ont probablement pesé sur la qualité des tests. Dans leur état actuel, ils ne peuvent pas remplacer les tests écrits à la main dans `MediaTypeTest`.



[^1]: `tika-core/src/test/java/org/apache/tika/mime/MediaTypeTest.java`
[^2]: `tika-core/target/site/jacoco/org.apache.tika.mime/index.html`
[^3]: `tika-core/target/pit-reports/org.apache.tika.mime/index.html`
[^4]: `tika-core/target/site/jacoco/org.apache.tika.mime/MediaType.html`
[^5]: `tika-core/target/pit-reports/org.apache.tika.mime/MediaType.java.html`
[^6]: `tika-core/src/main/java/org/apache/tika/mime/MediaType.java`
[^7]: `tika-core/rapports-chatunitest-initiaux/` (je pense pu que le path est accurate depuis que j’ai rejoué dans le code. Bref, à revoir)


# Tâche 2

## 1. Classe à tester:

### 1.1Classe retenue
Le choix de classes devait être basé sur la présence de tests pour la classe en question et une couverture de test inférieure à 100%. La classe __`MediaType.java`__, présente dans le module tika-core, a été choisie entre autres puisque son fichier de test existe[^1]. De plus, l'index[^2] de couverture avec JaCoCo indique une __couverture d'instructions de 79%__ et une couverture de branches de 74%. De plus, le _Pit Test Coverage Report_[^3] nous indique une __couverture de mutation de 54 %__, soit 46/85. En effet, malgré une couverture d'instruction de 79 %, 46 mutants sont tués, __6 mutants survivent__ et 33 mutants (38,82 %) n'ont pas pu être détectés par manque de tests adéquats exécutant le code correspondant à chaque mutant. Les mutants survivants montrent que certains tests ont réussi malgré la modification introduite par PIT, ce qui indique ainsi des comportements qui ne sont pas suffisamment vérifiés par les tests. Cela nous permet d'avoir des pistes sur les améliorations ou les ajouts à faire pour améliorer les tests existants. 

La classe `MediaType.java` comporte 28 méthodes dont 11 avec une couverture inférieure à 100%.
<img width="989" height="659" alt="image" src="https://github.com/user-attachments/assets/f44fa766-0859-4e8b-a8ba-f2b532704aef" />[^4]

### 1.2 Méthode retenue
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

## 2. IA et test

### 2.1 Installation et environnement local

ChatUniTest est un plugin Maven qui génère des tests JUnit à l'aide d'un modèle de langage (LLM). Nous l'avons intégré au module `tika-core`, dans le même `pom.xml` que PIT, pour générer des tests pour la classe `org.apache.tika.mime.MediaType`.

Deux éléments ont été ajoutés à `tika-core/pom.xml`, comme le demande la documentation du plugin :

1. __La dépendance `chatunitest-starter` 1.4.0__ (`io.github.ZJU-ACES-ISE`, type `pom`), qui fournit les bibliothèques utilisées par les tests générés (JUnit, Mockito). Nous l'avons placée dans la portée `test` pour qu'elle ne soit pas transmise aux projets qui dépendent de `tika-core`.
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

### 2.2 Défis rencontrés lors de l'installation de chatUnitest

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

## 3. Génération et évaluation des tests

### 3.1 Emplacements des tests

ChatUniTest a écrit 9 fichiers dans `tika-core/chatunitest-tests/org/apache/tika/mime/` : 8 classes de test, nommées `MediaType_<méthode>_<n>_<n>_Test.java`, et `MediaType_Suite.java`, qui les regroupe. Elles visent 8 des 17 méthodes publiques de `MediaType` : `application`, `audio`, `image`, `text`, `parse`, `equals`, `compareTo` et `hasParameters`. Aucune classe n'a été produite pour `video`, `set` (deux variantes), `getType`, `getSubtype`, `getBaseType`, `getParameters`, `hashCode` et `toString`. 
Le dépôt contient donc les tests générés à deux endroits :

| Emplacement | Contenu |
|---|---|
| `tika-core/chatunitest-tests/org/apache/tika/mime/` | Les 9 fichiers Java générés par ChatUniTest, conservés séparément des tests du projet. Ce dossier contient les 8 classes de test générées ainsi que MediaType_Suite.java. |
| `tika-core/src/test/java/org/apache/tika/mime/` | Les 8 mêmes classes, corrigées pour que tous leurs tests réussissent. C'est cette version que Maven exécute. Le fichier `MediaType_Suite.java` n'y est pas. |

ChatUniTest permet de choisir son dossier de sortie (paramètre `testOutput`) ou de copier les tests générés dans `src/test` (commande `chatunitest:copy`). Nous avons gardé le dossier séparé, parce que les tests générés, placés tels quels dans `src/test`, font échouer le build :

- MediaType_Suite.java ne compile pas avec le classpath de tika-core : les imports org.junit.platform.runner.JUnitPlatform et org.junit.platform.suite.api.SelectClasses ne sont pas résolus. Une classe de test qui ne compile pas dans les sources de test intégrées peut bloquer la compilation de l'ensemble des tests du module.
- Sur les 95 tests générés, 48 échouent et 19 produisent une erreur, soit 67 tests qui ne réussissent pas. Maven arrête le build dès qu'un test échoue.
- Les fichiers générés n'ont pas l'en-tête de licence Apache attendu et utilisent des imports en étoile qu'on retrouve dans les huit classes de test en plus d'imports statiques comme `import static org.junit.jupiter.api.Assertions.*` et des imports classiques comme `import org.mockito.*`. Le build de Tika applique Checkstyle aux tests et refuse ces cas.

Générer directement dans `src/test` n'aurait donc évité aucune correction. Le dossier séparé conserve en plus la sortie brute de l'IA, sans laquelle on ne pourrait plus vérifier combien de tests échouaient ni combien de corrections ont été nécessaires.

### 3.2 Corrections et intégration des tests générés

#### Résultats des tests

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

Deux types de corrections ont été nécessaires : des ajustements techniques pour intégrer et exécuter les tests dans le projet, puis des modifications du contenu de certains tests générés pour faire réussir les 95 tests. À l’état brut, 67 exécutions de tests générés n’aboutissaient pas : 48 échecs et 19 erreurs. 

Seules les modifications qui empêchent le test de réussir seront apportées. Chaque résultat attendu corrigé a été déterminé à partir du code de `MediaType`.

| Classe | Méthodes corrigées | Nature des corrections |
|---|---:|---|
| `MediaType_application_0_0_Test` | 6 sur 8 | Les entrées utilisent des sous-types (`json`, `plain`, etc.) au lieu d'un type complet (`application/json`). Le type est vérifié sur la forme complète, le sous-type attendu n'est plus une chaîne vide, la valeur d'un paramètre est `utf-8` et non `charset=utf-8`, et la forme textuelle de la table des paramètres est `{param1=value1, param2=value2}`. Pour `null`, le résultat attendu est `application/null` et non `null`. |
| `MediaType_audio_1_2_Test` | 4 sur 5 | Le paramètre est placé dans la chaîne (`mp3; custom=value`) au lieu d'être passé comme second argument. Les résultats attendus sont corrigés pour `null` (`audio/null`), pour la chaîne vide (`null`) et pour `invalid` (`audio/invalid`). Le mock inutilisé et l’initialisation de Mockito sont retirés. |
| `MediaType_compareTo_20_1_Test` | 3 sur 5 | Les tests vérifient le signe du résultat (`> 0`) au lieu de la valeur exacte `-1` ou d'une `NullPointerException`. |
| `MediaType_equals_18_1_Test` | 0 sur 11 | Aucune correction. |
| `MediaType_hasParameters_15_3_Test` | 10 sur 10 | Le mock est remplacé par un vrai objet `MediaType` construit avec les mêmes paramètres. Pour le cas « paramètres nuls », qui ne peut pas exister dans la classe, le test utilise un type sans paramètre. |
| `MediaType_image_2_1_Test` | 8 sur 9 | La valeur d'un paramètre est `utf-8` et non `charset=utf-8`. Les 7 tests qui attendaient `null` pour une entrée jugée invalide attendent le type réellement retourné, par exemple `image/invalid`. |
| `MediaType_parse_7_0_Test` | 7 sur 7 | `@InjectMocks` et `@Mock` sont retirés, et `parse()` est appelée comme méthode statique. L'attribut `nullValues = "null"` est ajouté à `@CsvSource`, sans quoi le mot `null` est lu comme une chaîne. `parse(null)` est attendu à `null` et non à une exception. Les formes attendues sont les formes normalisées (sans guillemets) et la valeur `utf-8` telle qu'elle est donnée. |
| `MediaType_text_3_1_Test` | 0 sur 11 | Aucune correction. |
| __Total__ | __38 sur 66__ | |

Les huit classes ont également été mises en conformité avec les exigences du build de Tika : ajout de l’en-tête de licence Apache, nettoyage des imports et respect de la limite de 180 caractères par ligne.

Après ces corrections, les 95 tests de la version corrigée réussissent et les violations de Checkstyle sont éliminées. Ce résultat confirme que les tests peuvent être intégrés au projet, mais ne garantit pas à lui seul leur qualité fonctionnelle.

### 3.3 Évaluation des tests générés

#### 3.3.1 Défauts observés et critique des oracles

Cette section explique ce que chaque classe générée cherche à vérifier, puis critique la qualité des tests à partir du code de `MediaType`[^6]. Il faut savoir que les méthodes `application()`, `audio()`, `image()` et `text()` suivent toutes le même principe : elles ajoutent un préfixe au sous-type reçu et appellent `parse()`. Par exemple, `image("jpeg")` retourne `parse("image/jpeg")`.

| Classe de test | Ce que la classe vérifie | Réussis | Critique principale |
|---|---|---:|---|
| `MediaType_application_0_0` | `application()` avec des types valides, avec paramètres, invalides, nuls et vides | 2 / 28 | Passe un type complet (`application/json`) au lieu d'un sous-type (`json`) |
| `MediaType_audio_1_2` | `audio()` avec un type valide, nul, vide et invalide | 1 / 5 | Utilise inutilement la réflexion pour appeler une méthode publique, déclare un mock inutilisé et contient des attentes incorrectes pour plusieurs cas limites; un test cherche une surcharge `audio(String, Map)` inexistante. |
| `MediaType_compareTo_20_1` | L'ordre entre des types égaux, différents et `null` | 2 / 5 | Attend la valeur exacte -1 au lieu de vérifier le signe du résultat et attend à tort une NullPointerException lorsque les paramètres diffèrent; l'exception attendue pour un argument null est cohérente avec l'implémentation. |
| `MediaType_equals_18_1` | L'égalité selon le type, le sous-type et les paramètres | 11 / 11 | Valide, mais ne teste ni `null` ni un objet d'une autre classe |
| `MediaType_hasParameters_15_3` | `hasParameters()` avec zéro, un ou plusieurs paramètres | 0 / 10 | Teste un mock de `MediaType`, et non la vraie classe |
| `MediaType_image_2_1` | `image()` avec un sous-type valide, des paramètres et des entrées invalides | 1 / 9 | Suppose une validation qui n'existe pas |
| `MediaType_parse_7_0` | `parse()` avec des types simples, des paramètres et des valeurs entre guillemets | 0 / 16 | Aucun test ne s'exécute à cause de `@InjectMocks` |
| `MediaType_text_3_1` | `text()` avec des entrées jugées invalides | 11 / 11 | Réussit par accident; 6 tests identiques; aucun cas valide |

Les huit classes de tests générées par ChatUniTest ont été exécutées individuellement, sans modifier leur contenu. D'après les rapports[^7], sur les 95 tests exécutés, 28 ont réussi, 48 ont échoué et 19 ont provoqué des erreurs d’exécution. Un test qui réussit n’est pas automatiquement bon, et inversement, un test qui échoue n’est pas automatiquement mauvais. Il faut vérifier les assertions et les comparer au comportement réel de MediaType. Les classes __`MediaType_equals_18_1_Test`__ et __`MediaType_text_3_1_Test`__ ont réussi tous leurs tests. 

Dans __`MediaType_application_0_0_Test`__, 26 des 28 tests ont échoué. Par exemple, `testNullApplication()` attendait null, alors que le résultat obtenu était application/null. D’autres tests, comme `testApplicationWithCharset()` et `testApplicationWithParameters()`, attendaient une valeur non nulle, mais la méthode retournait null pour les entrées utilisées. Ces résultats montrent que les cas de test et les résultats attendus doivent être vérifiés à partir du comportement réel de la méthode `application()`.

Dans __`MediaType_compareTo_20_1_Test`__, trois des cinq tests ont échoué. Le test `testCompareToDifferentTypeSameParameters()` attendait exactement -1, alors que la méthode retournait 19. Or, la méthode `compareTo()` doit notamment respecter le signe correspondant à l'ordre de comparaison; elle ne garantit pas nécessairement une valeur exacte de -1 ou de 1. Deux autres tests attendaient une NullPointerException, mais aucune exception n'a été lancée. Ces assertions reposent donc sur des hypothèses qui ne correspondent pas au comportement observé.

Dans __`MediaType_image_2_1_Test`__, huit des neuf tests ont échoué. Par exemple, `testImageMethodWithInvalidType()` attendait null, alors que le résultat était image/invalid. De même, `testImageMethodWithParameters()` attendait charset=utf-8, alors que la valeur récupérée était utf-8. Ces exemples indiquent que certains tests supposent une validation plus stricte que celle effectuée par la méthode ou comparent incorrectement la valeur d'un paramètre.

Enfin, dans __`MediaType_hasParameters_15_3_Test`__, huit tests ont échoué et deux ont provoqué des erreurs. Plusieurs tests attendaient true pour des objets comportant des paramètres, mais obtenaient false. Deux autres tests ont échoué à cause de configurations Mockito jugées inutiles.

#### Défauts observés:

__1. Le modèle invente la spécification au lieu de lire le code.__ C'est la cause de la plupart des 48 échecs. Les tests supposent que `MediaType` valide ses entrées et retourne `null` pour un type inconnu, alors que le code accepte tout nom syntaxiquement correct : `image("invalid")` retourne `image/invalid`, et `application(null)` retourne `application/null`. De même, `MediaType_compareTo_20_1_Test` attend une `NullPointerException` quand les paramètres diffèrent, et `MediaType_parse_7_0_Test` attend une `IllegalArgumentException` pour `parse(null)`, alors que la méthode retourne `null`.

__2. Les méthodes sont mal utilisées.__ `MediaType_application_0_0_Test` appelle `application("application/json")`, ce qui donne `application/application/json`, que `parse()` rejette. Ses assertions sont aussi incohérentes : elles attendent que `getType()` retourne le type complet et que `getSubtype()` retourne une chaîne vide. `MediaType_equals_18_1_Test` écrit `new MediaType("text/plain", "utf-8")` en croyant fournir un type et un jeu de caractères, alors que le second argument du constructeur est le sous-type.

__3. Mockito et la réflexion sont utilisés sans raison, et cassent des tests.__ `MediaType` est une classe de valeur sans dépendance : il n'y a rien à simuler. Pourtant :
- `MediaType_hasParameters_15_3_Test` appelle `hasParameters()` sur un mock. C'est Mockito qui répond `false`, et le code de `MediaType` n'est jamais exécuté. Même réussis, ces 10 tests ne vérifieraient rien.
- `MediaType_parse_7_0_Test` annote un champ `MediaType` avec `@InjectMocks`. Mockito ne peut pas l'instancier, et les 16 tests tombent en erreur avant de s'exécuter.
- `MediaType_audio_1_2_Test` appelle `audio()` par réflexion, avec le commentaire « private audio method », alors que la méthode est publique. Un des tests lui passe deux arguments au lieu d'un, d'où l'erreur `wrong number of arguments`.

__4. Des tests réussissent sans rien démontrer.__ Les 11 tests de `MediaType_text_3_1_Test` passent tous, mais pour une mauvaise raison. Ils appellent par exemple `text("text/plain; charset=utf-8; invalid")`, ce qui donne `text/text/plain; ...` : le résultat est `null` à cause du préfixe en double, et non à cause du paramètre « invalide » que le test croit vérifier. Aucun test ne vérifie le cas normal, `text("plain")`. C'est un exemple concret du fait qu'un test qui réussit n'est pas forcément un bon test.

__5. Les tests sont redondants et mal nommés.__ Dans `MediaType_text_3_1_Test`, 6 des 11 tests utilisent exactement la même entrée sous des noms différents (`...InvalidKeyAndValue`, `...InvalidKeyAndValue2`, `...3`, `...4`). Dans `MediaType_equals_18_1_Test`, 9 des 11 tests vérifient la même propriété (deux objets différents ne sont pas égaux) avec des combinaisons très proches, et `testEqualsWithDifferentOrderOfParameters` n'échange pas l'ordre des paramètres : il inverse la clé et la valeur. Le nombre de tests donne donc une fausse impression de couverture.

__6. Absence de ciblage délibéré des mutants initiaux.__ Les mutants survivants se trouvent dans `isSimpleName()`, accessible par `parse()`. Les tuer demande des entrées aux limites des caractères acceptés (par exemple une majuscule, un espace, ou les caractères `-`, `+`, `.` et `_`). Aucun test généré ne cible ces limites de façon délibérée, et la seule classe consacrée à `parse()` ne s'exécute pas. L'analyse de mutation nuance toutefois ce constat : ajoutés aux tests d'origine, les tests générés détectent 4 de ces 6 mutants, sans les viser, parce que plusieurs tests attendent `null` pour une chaîne mal formée. Les 2 mutants qui restent sont justement ceux qui demandent des entrées aux limites.

#### Points positifs et limites des tests corrigés

- Les 8 classes corrigées s'exécutent et suivent les conventions du projet (JUnit 5, même paquet que la classe testée).
- Les tests sont lisibles : noms descriptifs, structure préparation / action / vérification, tests paramétrés avec `@CsvSource` pour varier les entrées.
- `MediaType_equals_18_1_Test` est la classe la plus directement exploitable parmi les classes générées, ses assertions correspondent au comportement réel de `equals()`.
- Le modèle pense à des catégories de cas pertinentes (valeur nulle, chaîne vide, paramètres, guillemets), même si les résultats attendus sont souvent faux.

### Bilan

Sur 95 tests, 28 réussissent. Parmi eux, les 11 tests de `equals()` semblent vérifier des propriétés pertinentes, tandis que la valeur des autres tests doit être évaluée au cas par cas. Les tests générés par un petit modèle local (`qwen2.5-coder:3b`) peuvent servir de point de départ pour trouver des idées de cas, mais chaque résultat attendu doit être revérifié à partir du code. Ce constat est à lire avec les limites matérielles décrites plus haut (défi 6) : la taille du modèle et le peu de code qu'il pouvait lire ont probablement pesé sur la qualité des tests. Dans leur état actuel, ils ne peuvent pas remplacer les tests écrits à la main dans `MediaTypeTest`.

## 4. Analyse de mutation

### 4.1 Scores de mutation

Nous avons exécuté PIT sur `MediaType` avec les tests d'origine, puis avec les tests générés par ChatUniTest. PIT génère 85 mutants pour cette classe.

| Tests utilisés | Mutants tués | Non couverts | Survivants | Couverture de lignes |
|---|---:|---:|---:|---:|
| Tests d'origine (`MediaTypeTest`) | 46/85 (54 %) | 33 | 6 | 106/156 (68 %) |
| Tests générés seuls | 38/85 (45 %) | 38 | 9 | 76/156 (49 %) |
| Tests d'origine + tests générés | 59/85 (69 %) | 21 | 5 | 119/156 (76 %) |
| Tests générés corrigés seuls | 65/85 (76 %) | 16 | 4 | 124/156 (79 %) |
| Tests d'origine + tests générés corrigés | 65/85 (76 %) | 16 | 4 | 124/156 (79 %) |
| Tests d'origine + tests générés corrigés + tests manuels | 84/85 (99 %) | 0 | 1 | 156/156 (100 %) |

Les lignes « tests générés » portent sur la sortie brute de ChatUniTest, dont seuls les 28 tests qui réussissent sont pris en compte. Les lignes « tests générés corrigés » portent sur la version corrigée, dont les 95 tests réussissent. Le score des tests d'origine est celui du rapport PIT initial cité au début de ce document.

__Les tests générés ne détectent donc pas tous les mutants.__ Seuls et sans correction, ils en détectent moins que les tests d'origine. Ajoutés aux tests d'origine, ils font passer le score de 54 % à 69 %, et 26 mutants restent non détectés. Même une fois corrigés, ils laissent 20 mutants non détectés.

Conditions de mesure :

- Toutes les mesures sont faites sur une compilation propre par Maven (`clean` dans la même commande que PIT), comme dans la GitHub Action. Cette précaution est nécessaire : voir « Reproductibilité des mesures ».
- PIT refuse de démarrer si un test échoue. Comme 67 des 95 tests générés bruts échouent, nous avons activé l'option `skipFailingTests` de PIT pour cette version : seuls les 28 tests qui réussissent participent à l'analyse. Cette option n'est pas nécessaire pour la version corrigée.
- Pour la mesure sur la sortie brute, les 8 classes de `tika-core/chatunitest-tests/` ont été copiées temporairement dans `tika-core/src/test/java/org/apache/tika/mime/`, et `<targetTests>` a été réglé sur `org.apache.tika.mime.MediaType_*`.

### 4.2 Mutants détectés par les tests générés, et pourquoi

Les 38 mutants détectés par les tests générés bruts se répartissent en quatre groupes.

| Groupe | Mutants détectés | Test qui les détecte | Pourquoi le mutant est détecté |
|---|---:|---|---|
| `isSimpleName()` et le chemin rapide de `parse()` | 26 | `MediaType_text_3_1_Test` (tests `testTextInvalidParameters...` et `testTextAllWhitespaceType`), `MediaType_application_0_0_Test.testEmptyApplication` | Ces tests passent à `parse()` une chaîne qui n'est pas un nom simple, par exemple `text/text/plain; charset=utf-8; ...`, et attendent `null`. Un mutant qui rend `isSimpleName()` plus permissive (condition inversée, retour remplacé par `true`) fait accepter la chaîne : `parse()` retourne alors un objet, et `assertNull` échoue. |
| Retours de `parse()`, de `image()` et de `audio()` | 4 | `MediaType_image_2_1_Test.testImageMethod`, `MediaType_audio_1_2_Test.testAudioMethod` | Ces tests appellent la méthode avec un sous-type valide (`jpeg`, `mp3`) et comparent le résultat à une valeur précise. Un mutant qui remplace le retour par `null`, ou qui inverse le test `string == null`, change ce résultat. |
| Accesseurs `getType()`, `getSubtype()` et `toString()` | 4 | `MediaType_image_2_1_Test.testImageMethod` | Le test vérifie les trois valeurs exactes : `image`, `jpeg` et `image/jpeg`. Un retour remplacé par une chaîne vide, ou un indice décalé dans `getSubtype()` (`slash + 1` devenu `slash - 1`), donne une autre valeur. |
| `equals()` et le constructeur | 4 | `MediaType_equals_18_1_Test` | Les tests vérifient à la fois des cas d'égalité (`assertTrue`) et d'inégalité (`assertFalse`). Un mutant qui force le retour à `true` ou à `false`, ou qui inverse le test `instanceof`, fait échouer l'un des deux. Le mutant du constructeur (test `parameters.isEmpty()` inversé) est détecté par un test qui compare deux types avec paramètres. |

Deux observations :

- __Une seule classe détecte la majorité des mutants.__ Sur les 38 mutants détectés, 23 le sont par des tests de `MediaType_text_3_1_Test`. Or nous avons vu que ces tests réussissent pour une mauvaise raison : la chaîne est rejetée à cause du préfixe `text/` en double, et non à cause du paramètre que le test croit invalide. Ils détectent quand même des mutants, parce qu'ils exercent le code qui rejette une chaîne mal formée. Un test peut donc être utile au score de mutation tout en vérifiant autre chose que ce que son nom annonce.
- __Les détections viennent des oracles précis.__ `testImageMethod` détecte 7 mutants à lui seul parce qu'il compare plusieurs valeurs exactes. À l'inverse, un test qui n'attend que `null` ne peut détecter aucun mutant qui remplace un retour par `null`.

### 4.3 Mutants non détectés par les tests générés

Sur les 47 mutants que les tests générés bruts ne détectent pas, 38 se trouvent dans du code qu'aucun test généré réussi n'exécute, et 9 survivent.

__Mutants non couverts (38).__

| Méthode | Mutants | Raison |
|---|---:|---|
| `parseParameters()`, `unquote()` | 16 | Ces méthodes analysent les paramètres. La seule classe qui les vise, `MediaType_parse_7_0_Test`, ne s'exécute pas (erreur `@InjectMocks`), et les tests des autres classes qui passent des paramètres valides échouent. |
| `union()` | 7 | Aucun test généré n'utilise les constructeurs qui dérivent un type d'un autre. |
| `set()`, `getBaseType()`, `video()`, `hashCode()` | 9 | ChatUniTest n'a produit aucune classe de test pour ces méthodes. |
| `hasParameters()` | 2 | `MediaType_hasParameters_15_3_Test` contient 10 tests, mais ils appellent la méthode sur un mock : le code réel n'est jamais exécuté. |
| `parse()`, `equals()` | 4 | Branches non atteintes : trois retours de `parse()` (chaîne sans barre oblique, type analysé par le motif général, jeu de caractères avant le type) et la comparaison avec un objet d'une autre classe. |

__Mutants survivants (9).__

| Mutant | Raison de la survie |
|---|---|
| `application()` et `text()` : retour remplacé par `null` (2) | Les seuls tests réussis de ces deux méthodes attendent déjà `null`. Remplacer le retour par `null` ne change donc rien pour eux. |
| `compareTo()` : retour remplacé par `0` (1) | Les deux tests réussis vérifient l'égalité (résultat `0`) et l'exception sur `null`. Aucun ne vérifie un ordre. |
| `getParameters()` : retour remplacé par une table vide (1) | Le seul test réussi qui lit les paramètres attend justement une table vide. |
| Constructeur : calcul des indices et mise entre guillemets (3) | Les tests qui utilisent le constructeur ne vérifient que `equals()`, jamais `getType()`, `getSubtype()` ou la forme textuelle d'un paramètre. |
| `parse()` et `isSimpleName()` : chemin rapide (2) | Ces mutants ne changent que la mise en cache des types simples ou le traitement des types avec paramètres, qu'aucun test généré réussi n'observe. |

### 4.4 Apport des tests générés aux tests d'origine

Ajoutés aux tests d'origine, les tests générés bruts détectent 13 mutants que `MediaTypeTest` ne détectait pas, et le score passe de 46 à 59 :

- 4 des 6 mutants survivants de `isSimpleName()`, détectés par les tests qui attendent `null` pour une chaîne mal formée ou pour un nom vide;
- 9 mutants non couverts : les retours de `audio()` et de `image()`, `getType()`, les deux mutants de `getSubtype()`, les trois mutants de `equals()` et une branche de `parse()`.

Il reste 26 mutants non détectés avec les tests d'origine et les tests générés bruts réunis : 21 non couverts (`union()`, `set()`, `getBaseType()`, `hasParameters()`, `video()`, `hashCode()`, deux retours de `parse()` et une branche de `equals()`) et 5 survivants (`application()`, `text()`, `compareTo()`, et deux mutants de `isSimpleName()`).

### 4.5 Effet de la correction des tests générés

Une fois corrigés, les tests générés détectent 65 mutants au lieu de 38. Le gain vient surtout des classes qui ne s'exécutaient pas ou qui testaient un mock : `MediaType_parse_7_0_Test` couvre maintenant `parseParameters()` et `unquote()`, et `MediaType_hasParameters_15_3_Test` exécute enfin le code réel de `hasParameters()`. Les tests d'origine n'ajoutent alors plus rien : le score reste de 65 avec ou sans eux.

Avec les tests d'origine et les tests générés corrigés réunis, 20 mutants restent non détectés :

| Mutants restants | Nombre | Raison |
|---|---:|---|
| `union()`, `set()`, `video()`, `hashCode()` | 13 | Non couverts : aucun test généré ne vise ces méthodes ni les constructeurs dérivés. |
| `getBaseType()`, `parse()`, `equals()` | 3 | Non couverts : branches que les tests n'atteignent pas (type sans paramètre, jeu de caractères avant le type, comparaison avec un objet d'une autre classe). |
| `text()` : retour remplacé par `null` | 1 | Survivant : les 11 tests de `text()`, non modifiés, attendent tous `null`. |
| `isSimpleName()` : limite d'un intervalle et condition inversée | 2 | Survivants : ces mutants ne changent le résultat de `isSimpleName()` que pour certains caractères, et le seul effet visible est la mise en cache des types simples. Aucun test ne l'observe. |
| `parse()` : `return null` de la ligne 257 | 1 | Survivant : mutant équivalent, impossible à détecter (voir « Mutant équivalent »). |

Ce sont ces mutants que visent les tests écrits à la main, décrits dans la section suivante.

## 5. Tests écrits à la main

Nous avons écrit à la main une nouvelle classe de tests pour `MediaType`, dans un fichier séparé des tests d'origine et des tests générés :

`tika-core/src/test/java/org/apache/tika/mime/MediaTypeManualTest.java`

Elle contient 32 tests JUnit 5. Les tests d'origine (`MediaTypeTest`, 9 tests) n'ont pas été modifiés.

### 5.1 Démarche

Les tests ont été conçus à partir des rapports de mutation PIT, et non à partir de la seule couverture de lignes. L'analyse de mutation montre que les tests générés ne détectent pas tous les mutants : avec les tests d'origine, il en reste 26 non détectés pour la sortie brute, et 20 après correction. Nous avons écrit les tests manuels pour qu'ils complètent les tests d'origine sans dépendre des tests générés : ils visent les 39 mutants que `MediaTypeTest` seul ne détecte pas (33 non couverts et 6 survivants), ce qui couvre aussi les mutants que les tests générés laissent passer. Pour chacun de ces mutants, nous avons lu le code visé, déterminé le comportement attendu, puis écrit un test dont l'assertion échoue si ce comportement change. Chaque résultat attendu vient du code de `MediaType`[^6] et de sa Javadoc, pas d'une supposition.

### 5.2 Ce que les tests vérifient

| Groupe | Tests | Comportement vérifié | Mutants visés |
|---|---:|---|---|
| Méthodes de fabrique | 3 | `application()`, `audio()`, `image()`, `text()` et `video()` ajoutent leur préfixe au sous-type; un sous-type inconnu est accepté; un sous-type invalide donne `null` | Retour remplacé par `null` (5 mutants non couverts) |
| `set()` | 3 | Les deux variantes ignorent les valeurs nulles ou non analysables, fusionnent les doublons et retournent un ensemble non modifiable | Condition inversée, retour remplacé par un ensemble vide |
| `parse()` : entrées rejetées | 4 | `null`, absence de barre oblique, type ou sous-type vide, caractère réservé | Dernière ligne de `isSimpleName()` |
| `parse()` : noms simples | 4 | Un type simple est mis en cache; les caractères aux limites (`0`, `9`, `a`, `z`, `-`, `+`, `.`, `_`) sont simples; ceux juste à l'extérieur (`` ` ``, `{`, majuscules) ne le sont pas; le cache cesse d'accepter de nouveaux types à 10 000 entrées | Les 6 mutants survivants de `isSimpleName()` et la limite du cache |
| `parse()` : cas général | 2 | Espaces autour des noms; jeu de caractères placé avant le type (TIKA-350) | Branche `CHARSET_FIRST_PATTERN` |
| Constructeurs dérivés | 5 | Ajout, conservation, fusion et remplacement de paramètres; variantes avec un seul paramètre et avec un `Charset` | Les 7 mutants de `union()` |
| Accesseurs | 5 | `getType()`, `getSubtype()`, `getBaseType()`, `hasParameters()`; paramètres triés, en minuscules et non modifiables | Retours remplacés, addition remplacée par une soustraction |
| `equals()`, `hashCode()`, `compareTo()` | 6 | Égalité sur la forme normalisée; différences de type, de sous-type et de paramètres; `null` et autre classe; signe de `compareTo()` | Retours remplacés par `true`, `false` ou `0` |

### 5.3 Documentation de chaque cas de test

Pour chacun des 32 tests, les tableaux suivants donnent son nom, son intention (le comportement testé), la motivation des données choisies et l'explication de l'oracle (comment le comportement attendu a été déterminé). Les oracles viennent de deux sources : la Javadoc de `MediaType` quand elle décrit le comportement, et sinon la lecture du code de la méthode[^6].

#### 5.3.1 Méthodes de fabrique

| Test | Intention | Données et motivation | Oracle |
|---|---|---|---|
| `testFactoryMethodsPrefixSubtype` | Chaque méthode de fabrique ajoute son propre type devant le sous-type reçu. | Un sous-type courant et différent pour chacune des cinq méthodes (`pdf`, `mpeg`, `png`, `csv`, `mp4`), pour qu'une méthode qui utiliserait le préfixe d'une autre soit détectée. | Le code de chaque méthode est `parse("<type>/" + sous-type)`. Le résultat attendu est donc la chaîne exacte, par exemple `application/pdf`, comparée avec `toString()`. |
| `testFactoryMethodsAcceptUnknownSubtype` | Les fabriques ne valident pas le sous-type contre une liste de types connus. | `x-unknown-format`, un nom syntaxiquement valide qui ne correspond à aucun format réel. C'est exactement l'hypothèse fausse faite par les tests générés. | `parse()` n'accepte ou ne rejette une chaîne que sur sa syntaxe. Le nom est valide, donc le résultat attendu est `image/x-unknown-format`. |
| `testFactoryMethodsReturnNullForInvalidSubtype` | Un sous-type qui rend la chaîne complète invalide donne `null`. | Deux cas limites : le sous-type vide (`text/`) et un sous-type contenant une barre oblique (`text/a/b`). | La Javadoc de `parse()` indique que la méthode retourne `null` si l'analyse échoue. Un sous-type vide ou une deuxième barre oblique ne correspondent à aucun des motifs acceptés. |

#### `set()`

| Test | Intention | Données et motivation | Oracle |
|---|---|---|---|
| `testSetOfTypesSkipsNullsAndDuplicates` | `set(MediaType...)` ignore les valeurs nulles et ne garde qu'un exemplaire de chaque type. | Quatre arguments : deux types différents, une valeur `null` et un doublon du premier. Ils couvrent en un appel les deux branches du test `type != null`. | Le code n'ajoute que les types non nuls à un `HashSet`. On attend donc une taille de 2 et la présence des deux types. Vérifier la taille seule ne suffirait pas à détecter un ensemble contenant de mauvais éléments. |
| `testSetOfStringsSkipsUnparseableEntries` | `set(String...)` ignore les chaînes qui ne peuvent pas être analysées. | Deux chaînes valides, une chaîne invalide (`not a media type`) et `null`. Les deux dernières donnent `null` dans `parse()`, pour deux raisons différentes. | Le code n'ajoute un type que si `parse()` ne retourne pas `null`. On attend donc 2 éléments, égaux aux types `text/plain` et `image/png`. |
| `testSetIsUnmodifiable` | Les deux variantes retournent un ensemble que l'appelant ne peut pas modifier. | Un ensemble d'un élément pour chaque variante, auquel on tente d'ajouter un type. | La Javadoc des deux méthodes promet un ensemble non modifiable (« unmodifiable set »). On attend une `UnsupportedOperationException`. |

#### `parse()` : entrées rejetées

| Test | Intention | Données et motivation | Oracle |
|---|---|---|---|
| `testParseNullReturnsNull` | `parse(null)` retourne `null` au lieu de lancer une exception. | `null`, la valeur limite la plus évidente. Les tests générés supposaient ici une `IllegalArgumentException`. | Première ligne de `parse()` : `if (string == null) return null`. |
| `testParseWithoutSlashReturnsNull` | Une chaîne sans barre oblique n'est pas un type de média. | `text` (un seul nom) et la chaîne vide. | Le code retourne `null` quand `indexOf('/')` vaut -1, et aucun des deux motifs généraux n'accepte une chaîne sans barre oblique. |
| `testParseEmptyTypeOrSubtypeReturnsNull` | Un type ou un sous-type vide est rejeté. | Les trois combinaisons : type vide (`/plain`), sous-type vide (`text/`), les deux vides (`/`). Ce sont les seules entrées qui atteignent la dernière ligne de `isSimpleName()` avec un nom vide. | `isSimpleName()` se termine par `return name.length() > 0` : un nom vide n'est pas simple. Le motif général exige ensuite au moins un caractère de chaque côté de la barre. Le résultat attendu est donc `null`. |
| `testParseReservedCharacterReturnsNull` | Un caractère réservé dans un nom fait échouer l'analyse. | Un deux-points (`te:xt/plain`), un espace à l'intérieur d'un nom (`text/pl ain`) et une seconde barre oblique (`text/plain/extra`). Ces trois caractères font partie de ceux qu'exclut l'expression `VALID_CHARS`. | La Javadoc renvoie à la RFC 2045, et `VALID_CHARS` exclut ces caractères. Aucun motif ne correspond, donc `parse()` retourne `null`. |

#### `parse()` : noms simples et cache

| Test | Intention | Données et motivation | Oracle |
|---|---|---|---|
| `testParseSimpleTypeIsCached` | Un type composé de noms simples est mis en cache. | Un type `x-manual/<nom>` dont le nom n'a jamais été analysé (méthode `neverParsedName()`), pour que `isSimpleName()` soit réellement appelée à chaque exécution. | Pour un nom simple, `parse()` crée l'objet puis le range dans `SIMPLE_TYPES`. Deux appels avec la même chaîne doivent donc retourner le même objet (`assertSame`). La chaîne est aussi comparée à `toString()`. |
| `testParseSimpleNameBoundaryCharacters` | Les caractères situés aux bornes des intervalles acceptés sont bien des caractères simples. | Sept types construits avec les bornes des intervalles de `isSimpleName()` : chiffres `0` et `9`, lettres `a` et `z`, et les quatre symboles `-`, `+`, `.`, `_`. Ce sont les valeurs qu'un mutant de limite (`<=` devenu `<`) traiterait différemment. | `isSimpleName()` accepte `'0' <= c <= '9'`, `'a' <= c <= 'z'` et les quatre symboles. Ces noms sont donc simples, et le type doit être mis en cache : même objet au second appel. |
| `testParseStopsCachingAtLimit` | Le cache cesse d'accepter de nouveaux types quand il contient 10 000 entrées. | Le cache est rempli jusqu'à 9 999 entrées, juste sous la limite, puis deux nouveaux types sont analysés : le premier amène le cache à exactement 10 000, le second arrive quand la limite est atteinte. Ce sont les deux valeurs de part et d'autre de la borne. | La condition du code est `SIMPLE_TYPES.size() < 10000`. À 9 999, le type est mis en cache (même objet, taille 10 000). À 10 000, il ne l'est plus : deux appels donnent deux objets égaux mais distincts, et la taille reste 10 000. |
| `testParseNonSimpleNameIsNotCached` | Les caractères juste à l'extérieur des intervalles ne sont pas simples, mais restent valides. | L'accent grave et l'accolade ouvrante, qui précèdent `a` et suivent `z` dans la table ASCII, placés dans le type puis dans le sous-type. S'y ajoute un type en majuscules (`TEXT/Manual`). | Ces caractères ne sont pas dans les intervalles de `isSimpleName()`, mais `VALID_CHARS` les accepte. Le type est donc créé par le motif général, sans être mis en cache : deux appels donnent deux objets égaux mais distincts. Pour les majuscules, le constructeur convertit en minuscules, d'où `text/manual`. |

#### `parse()` : cas général

| Test | Intention | Données et motivation | Oracle |
|---|---|---|---|
| `testParseTrimsWhitespaceAroundNames` | Les espaces autour du type et du sous-type sont ignorés. | Une chaîne avec des espaces avant, après et autour de la barre oblique (`  text / plain  `). | Le motif `TYPE_PATTERN` autorise des espaces (`\s*`) à ces quatre endroits, et le constructeur applique `trim()`. Le résultat attendu est `text/plain`. |
| `testParseCharsetFirst` | Le jeu de caractères peut précéder le type (TIKA-350). | `charset=UTF-8; text/html`, la forme citée dans la Javadoc pour les serveurs web mal configurés. Aucun test existant n'atteignait cette branche. | La Javadoc de `parse()` indique que cette forme est acceptée. Le code construit le type à partir des groupes 2 et 3 du motif et le paramètre à partir du groupe 1. On vérifie la forme normalisée, le type, le sous-type et la table des paramètres. |

#### Constructeurs dérivés

| Test | Intention | Données et motivation | Oracle |
|---|---|---|---|
| `testDerivedTypeAddsParametersToPlainType` | Ajouter des paramètres à un type qui n'en a pas. | `text/plain` sans paramètre et une table d'un paramètre. C'est la première branche de `union()` (première table vide). | `union()` retourne la seconde table quand la première est vide. Le résultat attendu est `text/plain; charset=UTF-8`. |
| `testDerivedTypeKeepsParametersWhenNoneAdded` | Dériver un type sans ajouter de paramètre conserve ceux du type de base. | Un type avec un paramètre et une table vide. C'est la deuxième branche de `union()` (seconde table vide). | `union()` retourne la première table quand la seconde est vide. Le type obtenu doit avoir la même forme textuelle et être égal au type de base. |
| `testDerivedTypeMergesAndOverridesParameters` | Les paramètres des deux côtés sont fusionnés, et la nouvelle valeur remplace l'ancienne pour un même nom. | Un type de base avec `charset` et `format`, et une table qui redéfinit `charset` et ajoute `delsp`. Le nom en commun permet de vérifier lequel des deux `putAll` est appliqué en dernier. | `union()` copie la première table puis la seconde : la seconde l'emporte. Le constructeur trie ensuite les noms. On attend `charset=ISO-8859-1; delsp=yes; format=flowed` et 3 paramètres. |
| `testDerivedTypeWithSingleParameter` | Le constructeur `(type, nom, valeur)` ajoute un paramètre. | `text/plain` avec `format=flowed`. | La Javadoc indique que ce constructeur crée un type en ajoutant un paramètre au type de base. Résultat attendu : `text/plain; format=flowed`. |
| `testDerivedTypeWithCharset` | Le constructeur `(type, Charset)` ajoute le paramètre `charset`. | `text/html` avec `ISO_8859_1`, un jeu de caractères différent de l'UTF-8 utilisé ailleurs, pour ne pas réussir par coïncidence. | La Javadoc indique que ce constructeur ajoute le paramètre `charset`, et le code utilise `charset.name()`. Résultat attendu : `text/html; charset=ISO-8859-1`. |

#### Accesseurs

| Test | Intention | Données et motivation | Oracle |
|---|---|---|---|
| `testGetTypeAndSubtype` | `getType()` et `getSubtype()` excluent la barre oblique et les paramètres. | Un type sans paramètre dont le sous-type contient des points et un tiret (`application/vnd.ms-excel`), et un type avec paramètre. Le second vérifie que le sous-type s'arrête au point-virgule. | La Javadoc donne l'exemple : `text` et `plain` pour `text/plain`. Le code découpe la chaîne aux indices `slash` et `semicolon`. |
| `testGetBaseTypeWithoutParametersReturnsSameInstance` | Sans paramètre, `getBaseType()` retourne l'objet lui-même. | `image/png`, un type sans paramètre. | Première branche du code : `if (parameters.isEmpty()) return this`. On vérifie l'identité avec `assertSame`. |
| `testGetBaseTypeDropsParameters` | Avec paramètres, `getBaseType()` retourne le type sans ses paramètres. | `text/plain; charset=UTF-8`, l'exemple de la Javadoc. | La Javadoc donne `text/plain` pour `text/plain; charset=utf-8`. On vérifie l'égalité avec `TEXT_PLAIN`, la forme textuelle et l'absence de paramètres. |
| `testHasParameters` | `hasParameters()` distingue un type avec paramètres d'un type sans paramètre. | Un type sans paramètre et un type avec un paramètre : les deux valeurs possibles du résultat. Contrairement aux tests générés, la méthode est appelée sur de vrais objets. | La Javadoc indique `true` si le type a un ou plusieurs paramètres, `false` sinon. |
| `testGetParametersIsSortedAndUnmodifiable` | Les noms de paramètres sont en minuscules et triés, et la table est en lecture seule. | Trois noms donnés dans le désordre et avec des majuscules (`Zeta`, `alpha`, `Mid`). L'ordre d'insertion diffère de l'ordre alphabétique, pour distinguer un tri réel d'un ordre conservé par hasard. | La Javadoc promet une table triée et immuable, avec des noms en minuscules. On attend l'ordre `alpha`, `mid`, `zeta`, et une `UnsupportedOperationException` à l'écriture, pour un type avec paramètres et pour un type sans paramètre. |

#### `equals()`, `hashCode()` et `compareTo()`

| Test | Intention | Données et motivation | Oracle |
|---|---|---|---|
| `testEqualsComparesNormalizedForm` | Deux types sont égaux quand leur forme normalisée est la même, quelle que soit la façon de les construire. | Le même type obtenu par `parse()` et par le constructeur, ce dernier avec des majuscules et des espaces. On teste l'égalité dans les deux sens et avec lui-même. | `equals()` compare les chaînes internes, que le constructeur normalise (minuscules, espaces retirés). Les deux objets doivent donc être égaux. |
| `testEqualsDetectsDifferences` | Toute différence de type, de sous-type ou de paramètre rend deux types inégaux. | Quatre types qui ne diffèrent du type de référence que par un seul élément : absence de paramètre, sous-type, valeur du paramètre, casse de la valeur. | `equals()` compare des chaînes : la moindre différence donne `false`. Le constructeur ne met en minuscules que les noms des paramètres, pas leurs valeurs, d'où l'inégalité entre `UTF-8` et `utf-8`. |
| `testEqualsRejectsNullAndOtherClasses` | `equals()` retourne `false` pour `null` et pour un objet d'une autre classe. | `null` et la chaîne `"text/plain"`, dont le contenu est identique à la forme textuelle du type. Une comparaison fondée sur `toString()` serait ainsi détectée. | Le code retourne `false` quand l'argument n'est pas une instance de `MediaType`. C'est aussi ce qu'exige le contrat de `Object.equals()`. |
| `testHashCodeIsConsistentWithEquals` | Deux types égaux ont le même code de hachage, et le code dépend du contenu. | Deux types égaux construits différemment, puis deux types différents (`text/plain` et `text/html`). Le second couple détecte un `hashCode()` constant. | Le contrat de `Object.hashCode()` impose le même code pour deux objets égaux. Le code retourne `string.hashCode()`, qui diffère pour ces deux chaînes. |
| `testCompareToOrdersByStringForm` | `compareTo()` ordonne les types selon leur forme textuelle. | `application/pdf` et `text/plain`, comparés dans les deux sens, puis deux types égaux, puis un type et sa variante avec paramètre. | Le code retourne `string.compareTo(that.string)`. Le contrat de `Comparable` ne fixe que le signe du résultat, donc on vérifie `< 0`, `> 0` et `== 0`, jamais une valeur exacte comme `-1`. |
| `testCompareToNullThrows` | Comparer avec `null` lance une `NullPointerException`. | `null`. | Le contrat de `Comparable.compareTo()` demande une `NullPointerException` pour `null`, et le code accède à `that.string`. |

### 5.4 Résultats

| Mesure (PIT, classe `MediaType`) | `MediaTypeTest` seul | Avec `MediaTypeManualTest` |
|---|---:|---:|
| Tests | 9 | 41 |
| Couverture de lignes | 106/156 (68 %) | 156/156 (100 %) |
| Mutants tués | 46/85 (54 %) | 84/85 (99 %) |
| Mutants non couverts | 33 | 0 |
| Mutants survivants | 6 | 1 |

Les tests manuels détectent les 6 mutants survivants de `isSimpleName()` et 32 des 33 mutants non couverts. Le seul mutant restant est un mutant équivalent, qu'aucun test ne peut détecter (voir ci-dessous). Les tests manuels détectent donc tous les mutants détectables.

Les 32 tests réussissent, et le module ne présente aucune violation Checkstyle. Avec les tests d'origine et les tests générés corrigés, la classe `MediaType` est maintenant vérifiée par 136 tests, qui réussissent tous : 9 d'origine, 95 générés puis corrigés, et 32 écrits à la main. Pour que PIT prenne en compte les nouveaux tests, `<targetTests>` dans `tika-core/pom.xml` contient `MediaTypeTest`, `MediaType_*` et `MediaTypeManualTest`. Dans cette configuration, PIT tue 84 mutants sur 85, sur notre machine comme dans la GitHub Action.

Les mesures se reproduisent avec :

```
.\mvnw.cmd -pl tika-core -am test "-Dtest=MediaType*Test" "-Dsurefire.failIfNoSpecifiedTests=false"
.\mvnw.cmd -pl tika-core clean test-compile org.pitest:pitest-maven:mutationCoverage
```

#### 5.4.1 Mutant équivalent

Le mutant restant se trouve à la ligne 257 de `MediaType.java`, dans `parse()` :

```java
synchronized (SIMPLE_TYPES) {
    MediaType type = SIMPLE_TYPES.get(string);
    if (type == null) {
        int slash = string.indexOf('/');
        if (slash == -1) {
            return null;        // ligne 257
```

PIT y applique le mutateur « remplacer la valeur retournée par `null` ». Or cette instruction retourne déjà `null` : le mutant a exactement le même comportement que le code d'origine. C'est un mutant équivalent, et aucun test, quel qu'il soit, ne peut le faire échouer. Notre test `testParseWithoutSlashReturnsNull` exécute bien cette ligne, et il réussit avec ou sans le mutant.

PIT écarte normalement ce genre de mutant. Il ne le fait pas ici parce que l'instruction se trouve dans un bloc `synchronized` : dans le code compilé, la libération du verrou s'intercale entre le chargement de `null` et le retour (`aconst_null`, `aload_1`, `monitorexit`, `areturn`), et PIT ne reconnaît plus un retour de `null`.

Le score de 84 sur 85 est donc le maximum atteignable pour cette classe avec les mutateurs par défaut de PIT.

### Difficulté rencontrée : le cache statique de `parse()`

Plusieurs mutants de `isSimpleName()` n'ont qu'un seul effet observable : un type simple est rangé dans un cache statique, de sorte que deux appels de `parse()` avec la même chaîne retournent le même objet. Notre premier test vérifiait cela avec une chaîne fixe (`assertSame` sur deux appels), et ces mutants survivaient quand même.

La cause est que PIT exécute les mutants dans une même machine virtuelle. Dès la première exécution du test, la chaîne fixe entre dans le cache; aux exécutions suivantes, `parse()` la retrouve dans le cache et ne rappelle plus `isSimpleName()`. Le mutant n'est donc jamais exercé. Nous avons corrigé le test pour qu'il utilise à chaque exécution un nom jamais analysé auparavant (méthode `neverParsedName()`), après quoi ces mutants sont tués.

Cette difficulté illustre un piège général : un test qui dépend d'un état statique peut réussir pour une raison étrangère à ce qu'il prétend vérifier, et seul le test de mutation nous l'a révélé.

### 5.5 Comparaison des oracles : tests générés et tests écrits à la main

Un oracle est la partie d'un test qui décide si le comportement observé est correct : ici, les assertions. Nous comparons les oracles produits par ChatUniTest à ceux écrits à la main, c'est-à-dire ceux de `MediaTypeTest` (tests d'origine de Tika) et de `MediaTypeManualTest` (nos tests), selon trois critères : la pertinence, la spécificité et la présence de vérifications triviales.

### 5.5.1 Vue d'ensemble

| Type d'assertion | Tests générés (8 classes) | `MediaTypeTest` | `MediaTypeManualTest` |
|---|---:|---:|---:|
| `assertEquals` | 42 | 28 | 41 |
| `assertNull` | 23 | 0 | 11 |
| `assertNotNull` | 7 | 11 | 0 |
| `assertTrue` / `assertFalse` | 21 | 1 | 12 |
| `assertThrows` | 4 | 0 | 5 |
| `assertSame` / `assertNotSame` / `assertNotEquals` | 0 | 0 | 12 |

Les tests générés contiennent autant d'assertions que les tests manuels, mais le nombre ne dit rien de leur valeur : sur les 95 tests générés, 67 échouent parce que leur oracle est faux ou ne s'exécute pas.

### Pertinence : l'oracle correspond-il au comportement réel ?

C'est la différence la plus nette. Les oracles manuels sont déduits du code et de la Javadoc; ceux de l'IA sont déduits de ce que le nom de la méthode laisse supposer.

| Comportement | Oracle généré par l'IA | Oracle écrit à la main |
|---|---|---|
| Sous-type inconnu | `assertNull(MediaType.image("invalid"))` : faux, la classe ne valide pas les sous-types. | `assertEquals("image/x-unknown-format", MediaType.image("x-unknown-format").toString())` |
| `parse(null)` | `assertThrows(IllegalArgumentException.class, () -> mediaType.parse(null))` : faux, la méthode retourne `null`. | `assertNull(MediaType.parse(null))` |
| Ordre de deux types | `assertEquals(-1, mediaType1.compareTo(mediaType3))` : faux, le résultat est 19. | `assertTrue(pdf.compareTo(plain) < 0)` : seul le signe est garanti par le contrat. |
| Valeur d'un paramètre | `assertEquals("charset=utf-8", imageType.getParameters().get("charset"))` : confond la valeur et le couple nom=valeur. | `assertEquals(Collections.singletonMap("charset", "UTF-8"), type.getParameters())` |
| Présence de paramètres | `when(mockMediaType.getParameters()).thenReturn(parameters); assertTrue(mockMediaType.hasParameters())` : l'oracle porte sur un mock. | `assertTrue(MediaType.parse("text/plain; charset=UTF-8").hasParameters())` |

Les oracles de l'IA ne sont pas tous faux : ceux de `MediaType_equals_18_1_Test` sont corrects et proches de ce que nous avons écrit pour `equals()`. Ils restent toutefois moins complets, car ils ne vérifient ni `null` ni un objet d'une autre classe.

### Spécificité : l'oracle distingue-t-il le bon résultat d'un résultat voisin ?

Un oracle spécifique échoue dès que le résultat s'écarte de la valeur attendue.

- __Les oracles manuels comparent des valeurs exactes.__ Par exemple, `testDerivedTypeMergesAndOverridesParameters` attend la chaîne complète `text/plain; charset=ISO-8859-1; delsp=yes; format=flowed`, ce qui vérifie à la fois la fusion, le remplacement et le tri des paramètres. Les tests du cache utilisent `assertSame` et `assertNotSame`, qui distinguent deux objets égaux de deux fois le même objet. Aucun oracle généré ne vérifie l'identité d'un objet, alors que c'est le seul moyen d'observer le chemin rapide de `parse()`.
- __Les oracles générés sont spécifiques dans quelques cas seulement.__ `testImageMethod` compare quatre valeurs exactes, et c'est le test généré qui détecte le plus de mutants à lui seul (7). À l'inverse, `MediaType_text_3_1_Test` n'utilise que `assertNull` dans ses 11 tests : il ne peut pas distinguer un rejet pour la bonne raison d'un rejet pour une autre raison, et c'est précisément ce qui s'y produit.
- __L'analyse de mutation confirme cet écart.__ Les tests générés qui réussissent détectent 38 mutants sur 85, et laissent survivre les mutants qui remplacent le retour de `text()` et de `application()` par `null`, puisque leurs oracles attendent déjà `null`. Les tests manuels, ajoutés aux tests d'origine, détectent 84 mutants sur 85, le dernier étant un mutant équivalent.

### Vérifications triviales

Une vérification triviale réussit presque quel que soit le comportement du code.

- __Dans les tests générés__, les plus fréquentes sont `assertNotNull(mediaType)` placé avant des assertions plus précises (7 occurrences), et surtout les assertions sur un mock : dans `MediaType_hasParameters_15_3_Test`, les 10 tests vérifient une valeur fournie par Mockito, et non par `MediaType`. Ces assertions ne peuvent révéler aucun défaut de la classe testée. S'y ajoute la répétition : 6 tests de `text()` appliquent le même oracle à la même entrée.
- __Les tests d'origine en contiennent aussi.__ `MediaTypeTest` compte 11 `assertNotNull`, dont plusieurs sur des valeurs qui ne peuvent pas être nulles, comme `assertNotNull(type.getParameters().keySet())`. Le test `testParseWithParams` vérifie que trois noms de paramètres sont présents, mais pas leurs valeurs. Un oracle écrit à la main n'est donc pas précis par nature.
- __Dans nos tests manuels__, nous n'avons gardé aucun `assertNotNull`. Quand un résultat non nul est attendu, il est comparé directement à sa valeur, ce qui échoue aussi sur une valeur nulle.

### 5.6 Bilan de la comparaison

| Critère | Oracles générés par l'IA | Oracles écrits à la main |
|---|---|---|
| Pertinence | Souvent faux : ils reflètent un comportement supposé (validation, exceptions, valeurs exactes de `compareTo()`). Corrects pour `equals()`. | Déduits du code et de la Javadoc; tous réussissent. |
| Spécificité | Inégale : précise dans `testImageMethod`, faible ailleurs (`assertNull` seul). | Élevée dans nos tests (valeurs exactes, identité des objets); moyenne dans `MediaTypeTest`. |
| Vérifications triviales | Fréquentes : `assertNotNull`, assertions sur des mocks, oracles dupliqués. | Absentes de nos tests; présentes dans `MediaTypeTest` (`assertNotNull`). |
| Mutants détectés | 38/85 pour les tests générés qui réussissent. | 46/85 pour `MediaTypeTest`, 84/85 avec `MediaTypeManualTest` (le mutant restant est équivalent). |

L'IA produit vite des oracles dont la forme est correcte, mais elle ne connaît pas le comportement réel de la classe : elle écrit ce que la méthode devrait faire d'après son nom. Un oracle écrit à la main demande de lire le code ou sa documentation, et c'est cette étape qui le rend pertinent. La comparaison avec `MediaTypeTest` montre aussi que cette lecture ne suffit pas : c'est l'analyse de mutation qui révèle si un oracle, humain ou généré, vérifie réellement quelque chose.


### 5.7 Reproductibilité des mesures

Pendant le travail, nos mesures locales de PIT ne correspondaient ni au rapport initial ni à la GitHub Action : nous obtenions par exemple 50 mutants tués au lieu de 46 pour les tests d'origine, et 85 sur 85 au lieu de 84 pour l'ensemble des tests.

La cause était la compilation. Sur le poste de développement, les classes de `tika-core/target/classes` avaient été produites par un autre compilateur que le `javac` du JDK 17 utilisé par la GitHub Action, et Maven les réutilisait sans les recompiler. Deux compilateurs sont possibles sur ce poste, et nous n'avons pas déterminé lequel : celui de l'extension Java de VS Code, qui compile en arrière-plan, ou celui du JDK 27, installé en premier et utilisé pour les toutes premières compilations. Un compilateur différent produit un code légèrement différent (157 lignes exécutables au lieu de 156), et PIT, qui travaille sur le code compilé, ne génère alors pas exactement les mêmes mutants.

En ajoutant `clean` à la commande de PIT, Maven recompile tout avec le `javac` du JDK 17. Les mesures locales sont alors identiques à celles de la GitHub Action, et le score des tests d'origine correspond exactement au rapport initial (46 tués, 33 non couverts, 6 survivants). Tous les chiffres de ce document ont été mesurés de cette façon.

### 5.8 Limites

- Le score de mutation porte sur les mutants que PIT génère avec ses mutateurs par défaut. Il ne prouve pas l'absence de défaut dans `MediaType`.
- Les tests du cache vérifient un détail d'implémentation (l'identité des objets retournés), et non un comportement promis par la Javadoc. Ils devraient être revus si le cache était retiré ou modifié.
- Le test de la limite du cache (`testParseStopsCachingAtLimit`) accède par réflexion au champ privé `SIMPLE_TYPES` pour remplir le cache jusqu'à 10 000 entrées, puis le remet dans son état d'origine. C'est le seul moyen d'atteindre cette limite sans perturber les autres tests, mais le test dépend du nom et du type de ce champ privé.
- Le cache statique rend aussi le verdict de PIT sensible à l'ordre d'exécution des mutants : un mutant peut laisser dans le cache une valeur qui influence les mutants suivants. Nous n'avons pas observé de variation entre deux exécutions sur une compilation propre, mais nous ne pouvons pas l'exclure.

## 6. Exécution dans la GitHub Action

Tous les nouveaux tests s'exécutent dans une GitHub Action ajoutée au dépôt :

- __Workflow :__ `tache 2 - tests MediaType`, défini dans `.github/workflows/tache2-tests-mediatype.yml`.
- __Déclenchement :__ à chaque push et à chaque pull request sur `main`, ou à la main depuis l'onglet Actions (bouton « Run workflow »).
- __Environnement :__ Ubuntu, JDK 17 (Temurin).

| Job | Commande | Résultat |
|---|---|---|
| `tests MediaType` | `./mvnw -B -pl tika-core -am test -Pci "-Dtest=MediaType*Test" -Dsurefire.failIfNoSpecifiedTests=false` | Réussi |
| `mutation PIT` | `./mvnw -B -pl tika-core test-compile org.pitest:pitest-maven:mutationCoverage` | Réussi : 84 mutants tués sur 85, couverture de lignes 156/156 |

Le premier job exécute les trois jeux de tests de `MediaType` : les tests d'origine, les tests générés corrigés et les tests écrits à la main. Le profil `ci` vérifie aussi le formatage du code et les règles Checkstyle.

Exécution de référence : https://github.com/ayaelb8/IFT3913_T2/actions/runs/38168580328

Pour vérifier le résultat, ouvrir cette exécution, puis le job `tests MediaType` et son étape « Tests de MediaType » : le journal Maven donne le nombre de tests exécutés par classe et se termine par `BUILD SUCCESS`. Les rapports détaillés sont joints à l'exécution : `rapports-surefire` pour les tests et `rapport-pit` pour l'analyse de mutation. Il faut être connecté à GitHub pour lire les journaux et télécharger les rapports.

Les autres workflows du dépôt (`main jdk17 build`, `main jdk21 build`, etc.) sont les workflows d'origine d'Apache Tika. Ils construisent l'ensemble du projet, ne font pas partie de cette tâche et ne fonctionnent pas forcément dans ce dépôt.

[^1]: `tika-core/src/test/java/org/apache/tika/mime/MediaTypeTest.java`
[^2]: `tika-core/target/site/jacoco/org.apache.tika.mime/index.html`
[^3]: `tika-core/target/pit-reports/org.apache.tika.mime/index.html`
[^4]: `tika-core/target/site/jacoco/org.apache.tika.mime/MediaType.html`
[^5]: `tika-core/target/pit-reports/org.apache.tika.mime/MediaType.java.html`
[^6]: `tika-core/src/main/java/org/apache/tika/mime/MediaType.java`

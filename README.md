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
[Expliquer comment ChatUniTest a été installé dans le pipeline Maven]

## Documentation des tests générés
### Problème de compilation avec la suite de tests générée

En intégrant les tests générés par ChatUniTest, j’ai également obtenu le fichier `MediaType_Suite.java`, qui regroupe les différentes classes de tests. Cependant, ce fichier utilise des imports JUnit Platform (`org.junit.platform.runner` et `org.junit.platform.suite.api`) qui ne sont pas disponibles dans la configuration actuelle du projet. Cela entraîne une erreur de compilation. Pour pouvoir évaluer les huit classes de tests individuellement sans modifier le contenu généré, j’ai déplacé le fichier `MediaType_Suite.java` hors du répertoire des tests pendant l'exécution des autres fichiers. Ce problème d’intégration a été noté séparément des échecs et des erreurs observés lors de l’exécution des tests eux-mêmes.

Dans `MediaType_audio_1_2_Test`, un test a également provoqué une erreur liée à un nombre incorrect d'arguments lors d'un appel par réflexion. Il s'agit là encore d'un problème dans la manière dont le test invoque la méthode.

### Résultats et analyse des tests générés

| Classe de test | Tests exécutés | Échecs | Erreurs |
|---|---:|---:|---:|
| `application_0_0` | 28 | 26 | 0 |
| `audio_1_2` | 5 | 3 | 1 |
| `compareTo_20_1` | 5 | 3 | 0 |
| `equals_18_1` | 11 | 0 | 0 |
| `hasParameters_15_3` | 10 | 8 | 2 |
| `image_2_1` | 9 | 8 | 0 |
| `parse_7_0` | 16 | 0 | 16 |
| `text_3_1` | 11 | 0 | 0 |
| **Total** | **95** | **48** | **19** |

Les huit classes de tests générées par ChatUniTest ont été exécutées individuellement, sans modifier leur contenu. D'après les rapports[^7], sur les 95 tests exécutés, 28 ont réussi, 48 ont échoué et 19 ont provoqué des erreurs d’exécution.

Un test qui réussit n’est pas automatiquement bon, et inversement, un test qui échoue n’est pas automatiquement mauvais. Il faut vérifier les assertions et les comparer au comportement réel de MediaType. Les classes `MediaType_equals_18_1_Test` et `MediaType_text_3_1_Test` ont réussi tous leurs tests. À l’inverse, plusieurs autres classes présentent des problèmes importants.

Dans `MediaType_application_0_0_Test`, 26 des 28 tests ont échoué. Par exemple, `testNullApplication()` attendait null, alors que le résultat obtenu était application/null. D’autres tests, comme `testApplicationWithCharset()` et `testApplicationWithParameters()`, attendaient une valeur non nulle, mais la méthode retournait null pour les entrées utilisées. Ces résultats montrent que les cas de test et les résultats attendus doivent être vérifiés à partir du comportement réel de la méthode `application()`.

Dans `MediaType_compareTo_20_1_Test`, trois des cinq tests ont échoué. Le test `testCompareToDifferentTypeSameParameters()` attendait exactement -1, alors que la méthode retournait 19. Or, la méthode `compareTo()` doit notamment respecter le signe correspondant à l'ordre de comparaison; elle ne garantit pas nécessairement une valeur exacte de -1 ou de 1. Deux autres tests attendaient une NullPointerException, mais aucune exception n'a été lancée. Ces assertions reposent donc sur des hypothèses qui ne correspondent pas au comportement observé.

Dans `MediaType_image_2_1_Test`, huit des neuf tests ont échoué. Par exemple, `testImageMethodWithInvalidType()` attendait null, alors que le résultat était image/invalid. De même, `testImageMethodWithParameters()` attendait charset=utf-8, alors que la valeur récupérée était utf-8. Ces exemples indiquent que certains tests supposent une validation plus stricte que celle effectuée par la méthode ou comparent incorrectement la valeur d'un paramètre.

Enfin, dans `MediaType_hasParameters_15_3_Test`, huit tests ont échoué et deux ont provoqué des erreurs. Plusieurs tests attendaient true pour des objets comportant des paramètres, mais obtenaient false. Deux autres tests ont échoué à cause de configurations Mockito jugées inutiles. Il faut donc vérifier à la fois la manière dont les objets de test sont construits et le comportement réellement défini par `hasParameters()`.



[^1]: `tika-core/src/test/java/org/apache/tika/mime/MediaTypeTest.java`
[^2]: `tika-core/target/site/jacoco/org.apache.tika.mime/index.html`
[^3]: `tika-core/target/pit-reports/org.apache.tika.mime/index.html`
[^4]: `tika-core/target/site/jacoco/org.apache.tika.mime/MediaType.html`
[^5]: `tika-core/target/pit-reports/org.apache.tika.mime/MediaType.java.html`
[^6]: `tika-core/src/main/java/org/apache/tika/mime/MediaType.java`
[^7]: `tika-core/rapports-chatunitest-initiaux/`


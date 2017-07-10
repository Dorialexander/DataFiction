# DataFiction
Ce répertoire Github héberge plusieurs bases de métadonnées sur la littérature française (généralement en provenance de la BNF) sous forme de fichiers csv.

Pour l'instant, il contient les jeux de données associés à la parution de l'article sur Sciences communes, "[Les femmes ont-elle disparu de la littérature en 1830 ?](https://scoms.hypotheses.org/824)" :
* *complete_novel_1680_1900.csv* (6 mo compressé, 26 mo décompressé) : ensemble des romans parus de 1680 à 1900. Cette recension intègre également les données du catalogue de la BNF (qui peuvent inclure des parutions non documentées sur Data BNF mais aussi des duplications).
* *persons_novel.csv* (5 mo compressé, 35 mo décompressé) : ensemble des "actes éditoriaux" associés à une publication. Ces actes éditoriaux correspondent à l'ensemble des rôles définis par la BNF (auteur du texte, traduction, illustrateur, etc.). Lorsqu'une publication a fait l'objet de plusieurs interventions, ses métadonnées sont répétées plusieurs fois.
* *persons_novel_alive.csv* (3,6 mo compressé, 23 mo décompressé) : même fichier que précédemment sauf que l'on n'a gardé que les auteurs vivants ou morts récemment au moment de la publication (et, donc, dont on connaissait l'état civil). C'est ce fichier qui été utilisé pour établir la part de romancières à différentes époques, afin de corriger partiellement les biais liés à la réédition de "classiques".

##Élaboration des fichiers

Tous ces fichiers ont été constitués avec les langages de programmation R et Python.  L'élaboration de *complete_novel_1680_1900.csv* et de *persons_novel.csv* a été assez complexe (avec quatre jointures successives sur Data BNF et le catalogue de la BNF) et fera l'objet d'une description ultérieure. Par contre *persons_novel_alive.csv* est juste un dérivé de *persons_novel.csv* généré par la commande R suivante :

```R
library(tidyverse)

persons_novel_alive <- persons_novel %>% 
  mutate(death_year = gsub("(^\\d\\d\\d\\d).*", "\\1", death)) %>% #On ne garde que les années pour les dates de décès (sans le mois ni le jour)
  mutate(death_year = as.numeric(death_year)) %>% #Transformation des années de mort en valeur numérique
  mutate(firstyear_data = as.numeric(firstyear_data)) %>% #Idem pour les années de parution
  filter(!is.na(death_year)) %>% #On retire toutes les valeurs qui ne sont pas valides
  filter(!is.na(firstyear_data)) %>% 
  mutate(diff_edition_death = firstyear_data-death_year) %>% #On définit la durée en année séparant la mort de la parution
  filter(diff_edition_death<5) %>% #On retire toutes les parutions postérieures à plus de 5 ans après la mort.
  filter(diff_edition_death>-90) #On retire également les parutions antérieures à 90 ans après la mort (quelques cas qui constituent manifestement des erreurs)
```

Pour créer le graphe de représentation des romancières nous avons utilisé les fonctions suivantes :

```R
library(tidyverse)

persons_novel_alive %>% 
  filter(language=="fre") %>% #Uniquement les œuvres en français (ailleurs les données de Gallica ne sont plus représentatives)
  filter(firstyear_data>1700) %>% #Les années de parution postérieures à 1700
  filter(role == "Auteur du texte") %>% #Uniquement les "auteurs du texte" (rôle 70 de la BNF)
  filter(sex != "_") %>% #On retire les auteurs de sexe indéterminé
  group_by(firstyear_data, sex) %>% #On groupe par année et sexe.
  summarise(count_author = n()) %>% #On récupère le nombre d'auteur homme et femme.
  mutate(freq = count_author/sum(count_author)) %>% #On calcule les fréquences par sexe.
  filter(sex == "female") %>% #On ne garde que les femmes
  ggplot(aes(firstyear_data, freq*100)) + geom_line(color="red") + labs(title="Taux de romancières dans Data BNF de 1700 à 1900", subtitle="Fréquences par groupe de dix ans", x="Année", y="Part des publications (en pourcentage)") + xlim(1700,1900) + ylim(0,60) #Création du graphe avec ggplot

```


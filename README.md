# Projet Python 2A

Projet Python de deuxième année pour le cours Python pour la Data Science réalisé par Audric Sicard et Eva-Andrée TIOMO.

## L'objectif de ce projet est d'observer les prix d'articles sur Vinted et étudier la corrélation de ces prix avec le niveau de revenus de la ville de vente ainsi que ses convictions écologiques globales.

## Introduction et motivations pour ce projet

Vinted est un site qui connait un fort succès depuis quelques années. Il s'agit d'un marché en ligne communautaire qui permet de vendre, acheter et échanger des vêtements et accessoires d'occasion. Alors que la fast fashion est de plus en plus décriée pour des raisons sociales et écologiques, il nous paraissait intéressant de nous pencher sur une solution au problème telle que Vinted. Notre projet a tenté de répondre à quelques interrogations : 
- Est-ce que les vêtements de seconde main sont plus chers selon la richesse de la commune de vente ?
- Est-ce que dans les régions qui votent le plus pour les écologistes il y a plus de ventes ?
- Est-ce que dans les régions qui votent le plus pour les écologistes il y a plus de ventes ?
  
Nous vous laissons lire la suite du projet pour en avoir les réponses.

## Étape 1 : extraction de données en scrappant le Vinted
### Scrapping
Le scrapper a été codé grâce au module selenium et fonctionne avec le browser chromedrivermanager.

Le scrapper commence par se connecter sur la page d'accueil du site Vinted.fr et par accepter les cookies s'ils existent. Il lance ensuite une recherche à partir des caractéristiques fournies par l'utilisateur sur le vêtement souhaité (exemple : tshirt noir homme ou encore jean homme), puis se dirige vers les résultats de recherches.
<img width="1391" alt="image" src="https://github.com/audricms/Vinted-pricer/assets/148848770/a8953e39-8d7b-4e48-8fde-ec2415654b11">

Ensuite, à partir de chaque article, on relève les informations suivantes : prix, marque, taille, état, matière, localisation, option de paiement, nombre de vues et date d'ajout.
Cependant selenium ne parvenait pas à obtenir les liens HTML de manière automatisée pour chacune de ces informations. Il a donc fallu identifier la structure des liens HTML (qui est plutôt homogène pour l'ensemble des articles) et obtenir le texte des liens : 
```
scrapping = {}
    
    for i in range(1,100) :
        try :
            XPATH = '/html/body/main/div/section/div/div[2]/section/div/div[2]/div[15]/div/div['+str(i)+']'
            x = browser.find_element(By.XPATH, XPATH)
            classx = x.get_attribute('class')
            if len(classx) > 15 :
                continue
        except NoSuchElementException:
            continue
        try : 
            XPATH = '/html/body/main/div/section/div/div[2]/section/div/div[2]/div[15]/div/div['+str(i)+']/div/div/div/div[2]/a'
            x = browser.find_element(By.XPATH, XPATH)
            scrapping[i] = x.get_attribute('href')
            scrapping2={}
        except NoSuchElementException:
            continue
```

Une fois que la structure du lien HTML correspondant à l'article est identifiée, on peut relever les informations mentionnées précédemment. 

Exemple de code pour trouver le prix d'un article et le stocker dans la colonne prix :
```
firstscrapping2={}

for i in firstscrapping.keys() :
    try:
        request_text = request.urlopen(firstscrapping[i]).read()
        page = bs4.BeautifulSoup(request_text, "lxml")
        firstscrapping2[i]={}
    
        prix = page.find('h1')
        if prix != None :
            firstscrapping2[i]['Prix (€)'] = prix.text[:-2]
        else : 
            firstscrapping2[i]['Prix (€)'] = float('nan')
```
La dernière étape est la création d'un dictionnaire avec toutes ces informations, pour tous les articles trouvés. Ce dictionnaire global est la concaténation de plusieurs dictionnaires correspondant chacun respectivement à une page.

### Interprétation des caractéristiques
On peut alors nettoyer le nom des marques, par exemple en regroupant les jeans de marque Levi Strauss & Co. et ceux de marque Levi's qui sont en réalité de la même marque.

### Sortie du scrapper
On obtient grâce au dernier nettoyage et la création du dictionnaire global, un gros dictionnaire qui nous servira de base de travail pour croiser les données socio-démographiques qu'on considère en plus par la suite.
On le retrouve en entier dans le document csv "jeanshomme.csv"

## Étape 2 : identification des villes de vente et étude de leurs revenus grâce à la base de données "revenus" ainsi que de leurs convictions écologiques grâce à la base de données "votes"

**I. Travail sur la base de données revenus**

Nous commençons par importer une base de données qui donne le nom de la commune, son code postal, le revenu moyen annuel des habitants et le revenu moyen annuel des habitants à l'échelle du département. Cette base de données a été trouvée sur le site data-gouv. 

Nous avons ensuite harmonisé les noms de colonnes et créé un petit dataframe comprenant une colonne "Département" et une colonne donnant le revenu moyen annuel par département associé.
Afin de visualiser plus clairement la répartition des revenus, nous avons utilisé le dataframe précédent pour obtenir une carte de la France indiquant les revenus par département.

Après avoir eu une idée globale, nous avons associé les villes à leur département pour passer à l'étape suivante : associer les votes écologiques aux villes.

**II. Travail sur la base de données des votes**

Nous avons obtenu sur le site data-gouv, une base de données détaillant les résultats de votes des législatives 2022 par département. Nous nous sommes concentrés sur les votes pour les écologistes ('ECO') et la Nouvelle union populaire écologique et sociale (NUP).

**Hypothèses :**

 - On suppose qu'une personne votant pour l'un des deux partis cités précédemment est plus sensible aux problématiques écologiques.
 - On suppose que le vote départemental est représentatif de la pensée par commune (ie si 20% du département vote Écologiste ou NUPES, on suppose que 20% de la ville étudiée appartenant au département aura eu ce même comportement). 
- On suppose qu'un département plus écologiques sera plus susceptible d'acheter des produits de seconde main (hypothèse qu'on testera à l'étape 3).

**Méthodologie**

_1. Nettoyage et filtration de la base de données_

 Nous avons téléchargé une base de données conséquente qui détaille par commune des informations sur le taux d'absentation mais précise aussi par département, le nomnbre de vote et les pourcentages pour chaque parti. Nous avons donc procédé de la manière suivante :
 
 - Renommer les colonnes en amont pour comprendre les données clairement %popent<sub> </sub>i correspond au pourcentage de la population totale en âge de voter qui a voté pour un parti<sub> </sub>i. %popvot<sub> </sub>i correspond au pourcentage de la population votante qui a voté pour un parti<sub> </sub>i.

- Supprimer les colonnes vides et les colonnes qui ne nous serviront pas dans la suite de l'analyse
  
- Sélectionner seulement le nombre de voix et les pourcentages pour le parti écologiste puis pour la NUPES. Comme les partis étaient dans la base de données dans différentes colonnes Parti1, Parti2, Parti3 et Parti4, il a fallu créer des dictionnaires intermédiaires avant de les concaténer et harmoniser les titres de colonnes.
  
_2. Compilation des votes pour avoir une idée des départements se souciant le plus de l'écologie_

Après avoir obtenu les dataframes correspondant à chaque parti, nous avons compilé les votes des deux partis par département pour essayer de définir grâce aux votes, les départements avec plus ou moins une sensibilité accrue aux problématiques environnementales.

## Étape 3 : agrégation  des données et définition des objectifs
Cette étape consiste simplement à agréger les sources de données entre elles. À ce stade, les données sont réparties en 3 dataframes que la phrase d'agrégation fusionne (`join`, `merge`, `set_index`) et nettoie (`rename`, `drop`). Ces 3 dataframes sont :

`jeanshomme` : c'est le dataframe obtenu à la fin de notre scrapping du site Vinted (voir étape 1).

`revenus` : c'est le dataframe obtenu à l'import des données de data-gouv sur les revenus par commune et département et après nettoyage (voir étape 2-I).

`votes` : c'est le dataframe obtenu à l'import des données de data-gouv sur les votes par département et après nettoyage (étape 2-II)

Exemple de manipulation pour obtenir notre dataframe final : 

```
df3['Prix (€)'] = df3['Prix (€)'].apply(lambda x: x.replace(',', '.'))
df3['Prix (€)'] = pd.to_numeric(df3['Prix (€)'])
df3 = df3.join(dféconupes.set_index('Code du département'), on = 'Département', how = 'left')

```
On obtient finalement le dataframe _"donnéesjointes.csv"_

### Définition des objectifs et des méthodes à partir de ce nouveau dataframe complet

- Comparer les deux cartes et voir si on peut observer des tendances similaires ou divergentes entre vote écologique et revenus sur selon les départements
- Faire des régressions linéaires pour répondre à nos interrogations initiales
- Conclure

## Étape 4 : affichage synthétisé des résultats et conclusions

- On obtient ces deux premiers histogrammes :
<img width="606" alt="image" src="https://github.com/audricms/Vinted-pricer/assets/148848770/3b64dd15-006c-48ed-815e-413941df81bf">


<img width="603" alt="image" src="https://github.com/audricms/Vinted-pricer/assets/148848770/78cd1694-1457-4927-8fb8-1d405e69e412">

Pour aller plus loin, on effectue deux régressions linéaires pour tester nos hypothèses sur le produit sélectionné. 
1. Une régression linéaire du prix sur plusieurs variables dont le niveau de vie de la commune, sa sensibilité écologique et la taille du vêtement.
2. Une régression linéaire linéaire du prix sur moins de variables mais qui comprend toujours le niveau de vie de la commune et sa sensibilité écologique.
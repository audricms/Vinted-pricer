# Projet Python 2A

*Ce projet est réalisé dans le cadre du cours de Python pour la Data Science de Lino Galiana par Audric Sicard et Eva-Andrée Tiomo.*

## L'objectif de ce projet est d'observer les prix d'articles sur Vinted.fr et étudier leur corrélation avec leurs caractéristiques, le niveau de revenus de la ville de et ses convictions écologiques globales.

## Introduction et motivations pour ce projet

Vinted.fr est un site qui connait un fort succès depuis quelques années. Il s'agit d'un marché en ligne communautaire qui permet de vendre, acheter et échanger des vêtements et accessoires d'occasion. Alors que la fast fashion est de plus en plus décriée pour des raisons sociales et écologiques, il nous paraissait intéressant de nous pencher sur une solution au problème telle que Vinted. Notre projet a tenté de répondre à quelques interrogations : 
- Est-ce que les vêtements de seconde main sont plus chers selon la richesse de la commune de vente ?
- Est-ce qu'il y a plus d'annonces de vente dans les zones géographiques qui votent proportionnellement plus pour les écologistes ?
  
Nous vous laissons lire la suite du projet pour en avoir les réponses.

## Structure du projet !!!!!

## Étape 1 : Récolte de données en scrappant le site Vinted.fr

Le scrapping des données a été réalisé dans le notebook **`VintedScrapping.ipynb`**.

### Scrapping
Le scrapper a été codé grâce au module `Selenium` qui fonctionne avec le browser **`Chronium`**, et le module `BeautifulSoup4`.
Le scrapper se résume à deux fonctions majeures, `h_creator`, qui convertit une chaîne de caractères correspondant à la recherche de l'utilisateur, un prix minimum, un prix maximum et le numéro de la page d'annonces, en un lien vers la page adéquate, et `tableau`, qui à partir dudit lien retourne un `DataFrame` contenant les annonces et leurs caractéristiques liées à la recherche de l'utilisateur. 
Le scrapper procède en ces deux étapes à la suite de difficultés rencontrées lors de l'automatisation du scrapping. Lors de la création automatisée, une erreur de concaténation entre une chaîne de caractère et un `WebElement` survenait. L'automatisation du scrapping est ainsi une première piste d'amélioration.

Pour ce projet, nous avons décidé de scrapper les 20 premières pages d'annonces de vente de jeans hommes par des particuliers. En effet, il s'agit d'un produit dont la durabilité est connue, souvent acheté de seconde main, avec a priori une certaine variabilité de prix, et donc propice à nos recherches.
Ainsi le scrapper se dirige dans un premier temps vers la page d'annonces demandée et garde en mémoire les liens url des annonces grâce à `Selenium`.

<img width="1391" alt="image" src="https://github.com/audricms/Vinted-pricer/assets/148848770/a8953e39-8d7b-4e48-8fde-ec2415654b11">

Exemple de code pour récupérer les liens url des annonces des particuliers à partir de la page d'annonces :

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
     for i in scrapping.keys() :
        time.sleep(1)
        try:
            request_text = request.urlopen(scrapping[i]).read()
            page = bs4.BeautifulSoup(request_text, "lxml")
            scrapping2[i]={}
```

Puis le scrapper s'appuie sur `BeautifulSoup4` pour accéder au code html desdites annonces et récolter les caractéristiques des annonces : prix, marque, taille, état, matière, localisation, option de paiement, nombre de vues et date d'ajout.

Au départ nous souhaitions seulement utiliser le module `Selenium` mais avons rencontré un certain nombre de difficultés, notamment une incapacité d'accéder au code html des annonces dans son éntièreté. C'est pourquoi nous nou ssommes aussi aidé de `BeautifulSoup4`. La structure homogène du code html des annonces a ensuite permis une automatisation facilitée de la récolte. 

Exemple de code pour accéder au code html d'une annonce et en extraire le prix s'il existe :

```
    for i in scrapping.keys() :
        time.sleep(1)
        try:
            request_text = request.urlopen(scrapping[i]).read()
            page = bs4.BeautifulSoup(request_text, "lxml")
            scrapping2[i]={}
    
            prix = page.find('h1')
            if prix != None :
                scrapping2[i]['Prix (€)'] = prix.text[:-2]
            else : 
                scrapping2[i]['Prix (€)'] = float('nan')

```

Toutes les informations ont ainsi été gardées en mémoire dans un dictionnaire.

### Nettoyage de la base de données scrappées
Le nettoyage de la base de données concernait surtout les marques. Nous avons regroupé les marques similaires (par exemple Levi's et Levi Strauss & Co.) et assigné une valeur `nan` aux marques comme "je ne sais pas", "sans marque", etc.

### Sortie du scrapper
Le dictionnaire nettoyé est ensuite convertit en `DataFrame` puis gardé en mémoire sous forme de fichier csv **`jeanshomme.csv`** dans **`Bases de données`** pour être utilisé par la suite dans nos recherches.

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
- Nous avons obtenu les deux cartes suivantes :

<img width="1054" alt="image" src="https://github.com/audricms/Vinted-pricer/assets/148848770/0ad7b85d-25b2-4aa4-9ac3-08f8dead1dc1">

<img width="1013" alt="image" src="https://github.com/audricms/Vinted-pricer/assets/148848770/ebfffb18-fc3e-48b7-96ea-5c5dfdcf8067">

- On obtient ces deux premiers histogrammes :
<img width="606" alt="image" src="https://github.com/audricms/Vinted-pricer/assets/148848770/3b64dd15-006c-48ed-815e-413941df81bf">


<img width="603" alt="image" src="https://github.com/audricms/Vinted-pricer/assets/148848770/78cd1694-1457-4927-8fb8-1d405e69e412">

Pour aller plus loin, on effectue deux régressions linéaires pour tester nos hypothèses sur le produit sélectionné. 
1. Une régression linéaire du prix sur plusieurs variables dont le niveau de vie de la commune, sa sensibilité écologique et la taille du vêtement.

<img width="690" alt="image" src="https://github.com/audricms/Vinted-pricer/assets/148848770/ebdde553-6119-4dc9-91b4-596199b40791">

2. Une régression linéaire linéaire du prix sur moins de variables mais qui comprend toujours le niveau de vie de la commune et sa sensibilité écologique.

<img width="587" alt="image" src="https://github.com/audricms/Vinted-pricer/assets/148848770/05016a3c-a17a-419e-993f-19885d995dba">
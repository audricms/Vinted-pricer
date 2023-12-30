# Projet Python 2A

*Ce projet est réalisé dans le cadre du cours de Python pour la Data Science de Lino Galiana par Audric Sicard et Eva-Andrée Tiomo.*

**L'objectif de ce projet est d'observer les prix d'articles sur Vinted.fr et étudier leur corrélation avec leurs caractéristiques, le niveau de revenus de la ville de et ses convictions écologiques globales.**

## Introduction et motivations pour ce projet

[Vinted.fr](https://www.vinted.fr/) est un site qui connait un fort succès depuis quelques années. Il s'agit d'un marché en ligne communautaire qui permet de vendre, acheter et échanger des vêtements et accessoires d'occasion. Alors que la fast fashion est de plus en plus décriée pour des raisons sociales et écologiques, il nous paraît intéressant de nous pencher sur une solution au problème telle que Vinted. Notre projet tente de répondre à quelques interrogations : 
- Est-ce que les vêtements de seconde main sont plus chers selon la richesse de la commune de vente ?
- Est-ce qu'il y a plus d'annonces de vente dans les zones géographiques qui votent proportionnellement plus pour les écologistes ?
  
Nous vous laissons lire la suite du projet pour en avoir les réponses.

### Structure du projet !!!!!

## Étape 1 : Récolte de données en scrappant le site [Vinted.fr](https://www.vinted.fr/)

Le scrapping des données est réalisé dans le notebook **`VintedScrapping.ipynb`**.

### Scrapping
Le scrapper est codé grâce au module `Selenium` qui fonctionne avec le browser **`Chronium`**, et le module `BeautifulSoup4`.
Le scrapper se résume à deux fonctions majeures, `h_creator`, qui convertit une chaîne de caractères correspondant à la recherche de l'utilisateur, un prix minimum, un prix maximum et le numéro de la page d'annonces, en un lien vers la page adéquate, et `tableau`, qui à partir dudit lien retourne un `DataFrame` contenant les annonces et leurs caractéristiques liées à la recherche de l'utilisateur. 
Le scrapper procède en ces deux étapes à la suite de difficultés rencontrées lors de l'automatisation du scrapping. Lors de la création automatisée, une erreur de concaténation entre une chaîne de caractère et un `WebElement` survenait. L'automatisation du scrapping est ainsi une première piste d'amélioration.

Pour ce projet, nous décidons de scrapper les 20 premières pages d'annonces de vente de jeans hommes par des particuliers. En effet, il s'agit d'un produit dont la durabilité est connue, souvent acheté de seconde main, avec a priori une certaine variabilité de prix, et donc propice à nos recherches.
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

Au départ nous souhaitions seulement utiliser le module `Selenium` mais rencontrions un certain nombre de difficultés, notamment une incapacité d'accéder au code html des annonces dans son éntièreté. C'est pourquoi nous nous sommes aussi aidés de `BeautifulSoup4`. La structure homogène du code html des annonces permet ensuite une automatisation facilitée de la récolte. Une piste d'amélioration serait alors de coder entièrement le scrapper avec `BeautifulSoup4`.

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

Toutes les informations sont ainsi gardées en mémoire dans un dictionnaire.

### Nettoyage de la base de données scrappées
Le nettoyage de la base de données concerne surtout les marques. Nous regroupons les marques similaires (par exemple Levi's et Levi Strauss & Co.) et assignons une valeur `nan` aux marques comme "je ne sais pas", "sans marque", etc.

### Sortie du scrapper
Le dictionnaire nettoyé est ensuite convertit en `DataFrame` puis gardé en mémoire sous forme de fichier csv **`jeanshomme.csv`** dans **`Bases de données`** pour être utilisé par la suite dans nos recherches.

## Étape 2 : Collecte de données sur les revenus et la coloration écologiste par commune en France métropolitaine

La collecte et la mise en forme de ces données de revenus et de coloration écologiste, comme leur aggrégation avec les données scrappées sont réalisées dans le notebook **`Revenus et votes.ipynb`**.

### Données sur les revenus par commune

Nous commençons par importer une base de données, `revenus.csv` qui donne le nom de la commune, son code postal, le revenu moyen annuel des habitants et le revenu moyen annuel des habitants à l'échelle du département en 2022. Cette base de données a été trouvée sur le site [data-gouv](https://www.data.gouv.fr/fr/).
A partir de cette base de données, nous créons un petit `DataFrame`, `dfrevdep`, qui contient la liste des départements identifiés par leur code postal, et leur revenu moyen annuel. Pour une meilleur visualisation, nous créons une carte de cette répartition.

<img width="1054" alt="image" src="https://github.com/audricms/Vinted-pricer/assets/148848770/0ad7b85d-25b2-4aa4-9ac3-08f8dead1dc1">

### Données sur la coloration écologiste par commune

Nous importons une base de données `votes.csv` trouvée sur [data-gouv](https://www.data.gouv.fr/fr/) qui détaille les résultats de votes des élections législatives de 2022 par département. Pour déterminer la coloration écologiste des départements, nous nous concentrons sur les votes pour le parti écologiste et pour la Nouvelle Union Populaire Ecologique et Sociale. Deux suppositions sont faites, voter pour l'un de ces deux partis c'est être plus sensible aux questions environnementales, et les résultats de votes à l'échelle du département reflètent ceux à l'échelle des communes. Une piste d'amélioration serait alors de pousser l'étude de la coloration écologiste par commune en prenant par exemple en compte des sondages d'opinion, ou en étudiant les votes par commune.
Pour créer un `DataFrame`, en l'occurrence `dféconupes`, avec les données voulues, nous commençons par supprimer en amont (sur la base de données téléchargée en local) les colonnes vides ou inutiles pour la suite de nos recherches, par exemple celle correspondant au taux d'abstention. Nous renommons les colonnes en amont pour une meilleure compréhension. Notamment, %popent<sub>i</sub> correspond au pourcentage de la population totale en âge de voter qui a voté pour un parti<sub>i</sub>. %popvot<sub>i</sub> correspond au pourcentage de la population votante qui a voté pour un parti<sub>i</sub>. Nous importons ensuite la base de données et en extrayons les votes pour le parti écologiste puis pour la NUPES par département. Pour cela, il a fallu extraire les données de votes, réparties dans des colonnes différentes dans la base de données, dans plusieurs `DataFrame`, avant de les concaténer pour créer respectivement `dféco` et `dfnupes`. Puis nous additionnons les votes pour ces deux partis dans le `DataFrame` `dféconupes`.

De même, nous créons une carte pour mieux visualiser la coloration écologiste par département.

<img width="1013" alt="image" src="https://github.com/audricms/Vinted-pricer/assets/148848770/ebfffb18-fc3e-48b7-96ea-5c5dfdcf8067">

### Aggrégation des bases de données

Pour mener l'étape de visualisation de nos données et de modélisation, nous joignons nos bases de données sur les annonces de jeans pour les hommes, sur les revenus par département et la coloration écologiste par département.
Avant la première jointure

Pour cela nous, lions chaque ville d'annonce de vente avec un son département par un `join` sur le noms des villes, formant le `DataFrame` `df3`. Puis nous y joignons le revenu moyen par département et les données de votes par deux `join` sur le code postal des départements.
Avant le premier 

Nous nettoyons ensuite `df3` en supprimant 




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

## Étape 3 bis : Visualisation 
À la suite du travail d'agrégation de données dans un `Dataframe` nommé `df3`, nous avons souhaité visualiser les corrélations entre les différentes variables afin de voir si une réponse à nos interrogations initiales se dessinait. Nous avons alors introduit `dfdesc` qui regroupe grâce à la commande groupby `df3` et `dféconupes`.

Nous avons d'abord cherché à exprimer le prix moyen des jeans hommes par département en fonction du pourcentage de votes pour un des partis écologistes (la NUPES, les Ecologistes). 
Pour ce faire, nous avons d'abord traité la colonne '%popent'grâce à la fonction `round2` qui arrondit les valeurs de la colonne à deux décimales et renvoie les chaînes de caractères telles quelles.
Nous avons ensuite trié le DataFrame en fonction de cette colonne grâce à la commande `sort_values` qui met ses valeurs dans l'ordre croissant. Cela est nécessaire pour que le graphique en barres soit bien ordonné. 
Enfin, nous avons créé un graphique en barres avec `Matplotlib` grâce à la commande `plot` et en utilisant les valeurs de '%popent' sur l'axe des x et les valeurs des prix moyens sur l'axe des y.

On obtient l'histogramme suivant :
<img width="606" alt="image" src="https://github.com/audricms/Vinted-pricer/assets/148848770/3b64dd15-006c-48ed-815e-413941df81bf">


Par la suite, nous avons souhaité observer le nombre d'annonces de jeans hommes par département en fonction du pourcentage de votes pour des partis écologistes. Notre objectif était de valider ou réfuter l'hypothèse selon laquelle un département plus écologique sera plus susceptible d'effectuer des transactions de produits de seconde main. 

Pour ce faire, nous avons repris `dfdesc` ainsi que le raisonnement précédent. Nous avons simplement changé l'axe des ordonnées pour y mettre le nombre d'annonces. 

On obtient l'histogramme suivant : 

<img width="603" alt="image" src="https://github.com/audricms/Vinted-pricer/assets/148848770/78cd1694-1457-4927-8fb8-1d405e69e412">


## Étape 4 : affichage synthétisé des résultats et conclusions
- Nous avons obtenu les deux cartes suivantes :



- On obtient ces deux premiers histogrammes :
<img width="606" alt="image" src="https://github.com/audricms/Vinted-pricer/assets/148848770/3b64dd15-006c-48ed-815e-413941df81bf">


<img width="603" alt="image" src="https://github.com/audricms/Vinted-pricer/assets/148848770/78cd1694-1457-4927-8fb8-1d405e69e412">

Pour aller plus loin, on effectue deux régressions linéaires pour tester nos hypothèses sur le produit sélectionné. 
1. Une régression linéaire du prix sur plusieurs variables dont le niveau de vie de la commune, sa sensibilité écologique et la taille du vêtement.

<img width="690" alt="image" src="https://github.com/audricms/Vinted-pricer/assets/148848770/ebdde553-6119-4dc9-91b4-596199b40791">

2. Une régression linéaire linéaire du prix sur moins de variables mais qui comprend toujours le niveau de vie de la commune et sa sensibilité écologique.

<img width="587" alt="image" src="https://github.com/audricms/Vinted-pricer/assets/148848770/05016a3c-a17a-419e-993f-19885d995dba">
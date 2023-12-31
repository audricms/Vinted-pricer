# Projet Python 2A

*Ce projet est réalisé dans le cadre du cours de Python pour la Data Science de Lino Galiana par Audric Sicard et Eva-Andrée Tiomo.*

**L'objectif de ce projet est d'observer les prix d'articles sur Vinted.fr et étudier leur corrélation avec leurs caractéristiques, le niveau de revenus de la ville de et ses convictions écologiques globales.**

## Introduction et motivations pour ce projet
[Vinted.fr](https://www.vinted.fr/) est un site qui connait un fort succès depuis quelques années. Il s'agit d'un marché en ligne communautaire qui permet de vendre, acheter et échanger des vêtements et accessoires d'occasion. Alors que la fast fashion est de plus en plus décriée pour des raisons sociales et écologiques, il nous paraît intéressant de nous pencher sur une solution au problème telle que Vinted. Notre projet tente de répondre à quelques interrogations : 
- Est-ce que les vêtements de seconde main sont plus chers selon la richesse de la commune de vente ?
- Est-ce qu'il y a plus d'annonces de vente dans les zones géographiques qui votent proportionnellement plus pour les écologistes ?
  
Nous vous laissons lire la suite du projet pour en avoir les réponses.

### Structure du projet
Le projet est structuré comme suit :
- **`VintedScrapping.ipynb`** est le notebook responsables du scrapping des données
- **`Revenus et votes.ipynb`** est le notebook responsables de la collecte des données non scrappées et de l'agrégation de toutes données
- **`Visualisation.ipynb`** est le notebook responsable de la visualisation
- **`Modélisation.ipynb`** est le notebook responsable de la modélisation
- **`Bases de données`** correspond au dossier de bases de données utilisées et créées au long du projet

Faire tourner les notebooks dans le même ordre que les étapes !

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

### Données sur la coloration écologiste par département
Nous importons une base de données `votes.csv` trouvée sur [data-gouv](https://www.data.gouv.fr/fr/) qui détaille les résultats de votes des élections législatives de 2022 par département. Pour déterminer la coloration écologiste des départements, nous nous concentrons sur les votes pour le parti écologiste et pour la Nouvelle Union Populaire Ecologique et Sociale. Deux suppositions sont faites, voter pour l'un de ces deux partis c'est être plus sensible aux questions environnementales, et les résultats de votes à l'échelle du département reflètent ceux à l'échelle des communes. Une piste d'amélioration serait alors de pousser l'étude de la coloration écologiste par commune en prenant par exemple en compte des sondages d'opinion, ou en étudiant les votes par commune.
Pour créer un `DataFrame`, en l'occurrence `dféconupes`, avec les données voulues, nous commençons par supprimer en amont (sur la base de données téléchargée en local) les colonnes vides ou inutiles pour la suite de nos recherches, par exemple celle correspondant au taux d'abstention. Nous renommons les colonnes en amont pour une meilleure compréhension. Notamment, %popent<sub>i</sub> correspond au pourcentage de la population totale en âge de voter qui a voté pour un parti<sub>i</sub>. %popvot<sub>i</sub> correspond au pourcentage de la population votante qui a voté pour un parti<sub>i</sub>. Nous importons ensuite la base de données et en extrayons les votes pour le parti écologiste puis pour la NUPES par département. Pour cela, il a fallu extraire les données de votes, réparties dans des colonnes différentes dans la base de données, dans plusieurs `DataFrame`, avant de les concaténer pour créer respectivement `dféco` et `dfnupes`. Puis nous additionnons les votes pour ces deux partis dans le `DataFrame` `dféconupes`.

De même, nous créons une carte pour mieux visualiser la coloration écologiste par département.

<img width="1013" alt="image" src="https://github.com/audricms/Vinted-pricer/assets/148848770/ebfffb18-fc3e-48b7-96ea-5c5dfdcf8067">

### Aggrégation des bases de données
Nous créons deux bases de données pour respectivement une visualisation et une modélisation futures par l'agrégation des données jusqu'alors collectées.

D'abord nous créons un `DataFrame` de mise en commun des données des annonces de jeans hommes, des résultats de votes par département et du revenu moyen par commune, `dfjointe`. Pour cela, on joint la ville d'annonce à son département et on supprime les lignes où la jointure n'a pas pu être faite, puis on joints les départements à leurs résultats de votes, puis au revenu moyen par commune. Puis on l'exporte sous format csv `donnéesjointes.csv` pour une future modélisation.

Ensuite nous créaons un `DataFrame` de mise en commun du prix moyen par département et de leur résultats de votes, `dfdesc`, que nous exportons sous format csv `desc.vsc` pour une visualisation future.

Exemple de manipulation pour obtenir notre dataframe final : 

```
dfjointe = df3.join(dfrev, on='Localisation', how='left')
```

## Étape 3 bis : Visualisation 
À la suite du travail d'agrégation de données dans un `Dataframe` nommé `df3`, nous avons souhaité visualiser les corrélations entre les différentes variables afin de voir si une réponse à nos interrogations initiales se dessinait. Nous avons alors introduit `dfdesc` qui regroupe `df3` et `dféconupes` grâce à la commande `groupby`.

Nous avons d'abord cherché à exprimer le prix moyen des jeans hommes par département en fonction du pourcentage de votes pour un des partis écologistes (la NUPES, les Ecologistes). 
Pour ce faire, nous avons d'abord traité la colonne '%popent'grâce à la fonction `round2` qui arrondit les valeurs de la colonne à deux décimales et renvoie les chaînes de caractères telles quelles.
Nous avons ensuite trié le DataFrame en fonction de cette colonne grâce à la commande `sort_values` qui met ses valeurs dans l'ordre croissant. Cela est nécessaire pour que le graphique en barres soit bien ordonné. 
Enfin, nous avons créé un graphique en barres avec `Matplotlib` grâce à la commande `plot` et en utilisant les valeurs de '%popent' sur l'axe des x et les valeurs des prix moyens sur l'axe des y.

On obtient l'histogramme suivant :

<img width="606" alt="image" src="https://github.com/audricms/Vinted-pricer/assets/148848770/3b64dd15-006c-48ed-815e-413941df81bf">


Par la suite, nous avons souhaité observer le nombre d'annonces de jeans hommes par département en fonction du pourcentage de votes pour des partis écologistes. Notre objectif était de valider ou réfuter l'hypothèse selon laquelle un département plus écologique sera plus susceptible d'effectuer un nombre de transactions plus élevé de produits de seconde main. 

Pour ce faire, nous avons repris `dfdesc` ainsi que le raisonnement précédent. Nous avons simplement changé l'axe des ordonnées pour y mettre le nombre d'annonces. 

On obtient l'histogramme suivant : 

<img width="603" alt="image" src="https://github.com/audricms/Vinted-pricer/assets/148848770/78cd1694-1457-4927-8fb8-1d405e69e412">


Enfin, nous nous sommes penchés sur le lien entre le revenu des départements et les prix des articles recherchés. Pour y parvenir, nous avons importé et nettoyé le petit DataFrame `revdep.csv` qui associe chaque département à son niveau de vie annuel moyen par foyer.
On utilise la fonction tonum pour transformer les chaînes de caractères en nombres quand elles sont propices à cette transformation afin d'harmoniser les numéros de département. 
On crée un DataFrame final qu'on nomme dfdesc2 et qui correspond à la fusion de dfdesc et dfdevrep grâce à la commande join.
Finalement, on utilise la même méthode que pour les deux histogrammes précédents appliquée cette fois-ci à dfdesc2. 

On obtient l'histogramme suivant : 
<img width="641" alt="image" src="https://github.com/audricms/Vinted-pricer/assets/148848770/e34b435f-fae9-47b2-8659-b85945a6dbb9">

On remarque au vu des trois histogrammes obtenus, que nos hypothèses de corrélation entre prix, nombre de votes, revenus et idéologie environnementale sont remises en question. En effet, il semble que ces paramètres n'influent pas particulièrement les uns sur les autres.

Ainsi, notre dernière partie a pour but d'observer les coefficients de corrélation de différentes variables dont celles mentionnées précédemment lorsqu'on effectue la régression linéaire du prix sur ces dernières. Nous chercherons à comprendre plus précisément les déterminants du prix.



## Etape 4 : Modélisation par régressions linéaires et interprétations
La modélisation est réalisée dans le notebook `Modélisation.ipynb`. Il s'agit de modéliser le prix d'une annonce de jean pour homme en fonction de certaines de ces caractéristiques. Pour cela nous calculons deux régressions linéaires.

### Création de la base de données d'intérêt
Nous préparons un `DataFrame`, `dfmodel`, qui réunit les données dont nous aurons besoin pour entraîner et tester le modèle, à savoir, le prix en euros, la taille, l'état, le nombre de vues, revenu de la commune de vente. Nous ne prenons pas en compte la commune puisque celle-ci est déjà représentée par son revenu moyen et les résultats de votes écologistes par département. Il en est de même pour le département. Nous ne prenons pas l'option de paiement en compte car c'est constamment par carte bleue. Enfin, une piste d'amélioration serait de prendre la marque en compte avec une base de données plus exhaustive des marques, et pour cela un plus grand nombre de d'annonces scrappées.


```
dfmodel = dfjointe[['Prix (€)', 'Taille', 'Etat', 'Vues', '%popvot', 'Niveau de vie commune']]
```

Puis nous transformons les variables catégorielles `Taille` et `Etat` en dummies.

Ensuite nous séparons la base de données en données d'entraîneement et de test. Une piste d'amélioration serait de scrapper plus d'annonces pour pouvoir stratifier cette séparation par commune. Dans notre cas, nous manquions de données et certaines strates en contenaient un trop petit nombre.

### Calcul de de la première régression linéaire
Nous standardisons et normalisons les données d'entraînement et de test et norme l2 pour une meilleure lecture des coefficients de la régression linéaire, puis nous la calculons. Il vient un R² de 0,26 aproximativement. La régression est donc faiblement prédictive du prix.

```
reg.score(X_test, Y_test)
```

### Visualisation des coefficients des variables expicatives de la première régression linéaire
Pour prendre connaissance des variables explicatives significatives (du point de vue de la prédiction), nous visualisons les coefficients de celles-ci dans la régression linéaire préalablement calculée.

<img width="665" alt="image" src="https://github.com/audricms/Vinted-pricer/assets/148848770/9e625f96-51b0-4e0f-a17b-fa8080f98d85">


Les variables de taille, en particulier 5XL et M semble être les plus significatives. L'une par sa rareté et l'autre son abondance sûrement. Aussi, le revenu par commune semble également significatif. L'état "Neuf avec étiquette" semble également significatif. Le résultat des votes est également moyennement significatif

### Calcul et visualisation des coefficients de la seconde régression linéaire
Par manque de données, nous soupçonnons que les variables de taille surentraîne le modèle. C'est poourquoi nous calculons une deuxième régression linéaire sans régresser le prix sur les variables de taille. Il vient un R² plus bas, d'approximativement 0,16. Il semblerait que notre hypothèse était mauvaise.

Aussi, nous visualisons les coefficients des variables explicatives de cette seconde régression sous la forme d'un barplot.

<img width="619" alt="image" src="https://github.com/audricms/Vinted-pricer/assets/148848770/0e59cec7-b7b1-4150-ab28-6ba6a0f4064c">


On remarque que les coefficients sont relativement inchangés en termes d'importance relative.

### Pistes d'amélioration de la modélisation
Il serait intéressante d'entraîner et de tester d'autres modèles. Le prix qui a priori aurait été une variable continue, est a en réalité plutôt discret. Il serait alors intéressant d'analyser les résultats de modèle de classification comme une forêt aléatoire.
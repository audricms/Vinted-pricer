# Projet Python 2A

Projet Python de deuxième année pour le cours Python pour la Data Science réalisé par Audric Sicard et Eva-Andrée TIOMO.

## L'objectif de ce projet est d'observer les prix d'articles sur Vinted et étudier la corrélation de ces prix avec la ville de vente et les convictions écologiques globales de cette dernière.

## Introduction et motivations pour ce projet

Vinted est un site qui connait un fort succès depuis quelques années. Il s'agit d'un marché en ligne communautaire qui permet de vendre, acheter et échanger des vêtements et accessoires d'occasion. Alors que la fast fashion est de plus en plus décriée pour des raisons sociales et écologiques, il nous paraissait intéressant de nous pencher sur une solution au problème telle que Vinted. Notre projet a tenté de répondre à quelques interrogations : 
- Est-ce que les vêtements de seconde main sont plus chers selon la richesse de la commune de vente ?
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
On obtient grâce au dernier nettoyage et la création du dictionnaire globale, un gros dictionnaire qui nous servira de base de travail pour croiser les données socio-démographiques qu'on considère en plus par la suite.
On le retrouve en entier dans le document csv "jeanshomme.csv"

## Étape 2 : identification des villes de vente et étude
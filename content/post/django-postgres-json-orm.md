---
title: "Django / PostGreSQL, l'ORM et le Json"
date: 2020-08-02T10:45:33+02:00
draft: false
tags: ["tutoriel", "theme"] 
---

# _Introduction_
Le **JSON** + le **relationnel** c'est carrement pratique !

Nous au boulot on en met
systématiquement dans chaque modèle (meme si on ne sait pas encore quoi en faire,
en général cela finit toujours par servir), le champs est appelé `data` ou `params`, un nom bien générique...
Cela nous permet d'allier le monde relationnel et le _NoSQL_(type documents).
Dans cet article, nous allons voir rapidemment comment utiliser le json avec Django. Nous verrons que Django possède quelques limites pour la manipulation
du JSON.

>L'exemple ci-dessous est tiré de la doc de django
```python
from django.contrib.postgres.fields import JSONField
 from django.db import models

  class Dog(models.Model):
      name = models.CharField(max_length=200)
      data = JSONField()
      params = JSONField()

      def __str__(self):
          return self.name
```

Et voila maintenant on peut mettre du json avec n'importe quel structure dans notre champs data. En Python l'intégration est transparent avec les dictionnaires.

### Problématique
```json
{
    "currency": "EUR",
    "elements": [
        {
            "price_components": [
                {
                    "vat": 20,
                    "type": "ENERGY",
                    "price": "0.0000",
                    "step_size": 1
                }
            ]
        }
    ]
}
```

Concretement ce json represente en tarif d'une recharge pour véhicule electrique.
Avec du json on peut faire plein de tarif du coup (la seul limite est notre imagination).

Dans l'exemple ci-dessous notre tarif est composé de 2 `price_components` :
- un prix sur l'energie avec un pas de 1
- un prix sur le temps de parking mais avec une restriction (cela signifie que le `price_component`
ne s'applique que si la durée de la recharge dépasse le min_duration)

```json
{
   "currency":"EUR",
   "elements":[
      {
         "price_components":[
            {
               "vat":20,
               "type":"ENERGY",
               "price":"0.0000",
               "step_size":1
            }
         ]
      },
      {
         "restrictions":{
            "min_duration":3660
         },
         "price_components":[
            {
               "vat":20,
               "type":"PARKING_TIME",
               "price":"0.0333",
               "step_size":60
            }
         ]
      }
   ]
}
```

Un json simple et complexe en meme temps, ce n'est pas nous qui l'avons inventé mais fait partie d'une norme...
nous on est disciplinés donc on appliquent...même si je sent la difficulté qui arrive...

Allez revenons à Django, maintenant que l'on a un super json à requêter

>Toutes les recharges avec une restrictions et toute les recharges avec la restrictions `PARKING_TIME`
```python
r = Recharge.objects.filter(price_ocpi__elements__1__has_key="restrictions")

r = Recharge.objects.filter(price_ocpi__elements__1__price_components__0__type="PARKING_TIME_ZW")
```
Cela fonctionne si l'on part du principe que les élements avec restrictions seront toujours à l'index 1

Bon ok mais, nous on veut quelque chose d'utile, comme par exemple avoir toutes les recharges gratuites ou payantes.

Cela se défini par l'addition de tous les **`price_components`**,
et si le résultat est égale à zero c'est une recharge gratuite, et > 0 c'est une recharge considéré comme payante.

Je part tête baisser,
pour faire un `.annotate` avec la fonction `Sum` et le problème sera réglé affaire suivante. Malheuresement cela ne c'est pas passé ainsi,
à la lecture de la doc django, et même du code source, j'ai compris que cela ne serais pas faisable avec l'ORM. Les fonctions sur le Json sont assez limitées

Nous avons deux solutions devant nous :
- soit en python à grand coup de map/filter mais dans ce cas il faut récuperer toutes les recharges (niveau optimisation et scalabilité on repassera)
- soit en SQL

J'ai choisis la solution SQL, car je ne suis pas fan de l'écriture et de la lecture de map/filter en python, mais aussi car
**PostgreSQL** possèdent un très bon support du **json** avec beaucoup de fonction très pratique.

---
Pour notre cas on va s'interesser à `jsonb_array_elements`, une fonction Postgres qui permet de décomposer un tableau json.

```sql
SELECT r.id, elements
FROM recharge r
jsonb_array_elements(price_ocpi -> 'elements') elements
where r.id = 10
```
Cette requête va nous retourner deux lignes (chaques objets de `elements`).
```json
10,"{""price_components"": [{""vat"": 20, ""type"": ""ENERGY"", ""price"": ""0.0000"", ""step_size"": 1}]}"
10,"{""restrictions"": {""min_duration"": 3660}, ""price_components"": [{""vat"": 20, ""type"": ""PARKING_TIME"", ""price"": ""0.0333"", ""step_size"": 60}]}"
```

---
Il ne reste plus car faire la somme des `price` des `price_components`
```sql
SELECT r.id, SUM((elements #> '{price_components, 0}' ->> 'price')::float)
FROM recharge r , jsonb_array_elements(price_ocpi -> 'elements') elements
where r.id = 10
group by r.id
having SUM((elements #> '{price_components, 0}' ->> 'price')::float) > 0.0
```

> Noter qu'avec **PostgreSQL 12**, il y a le nouveau langage pour parcourir du json
Nous aurions pu écrire la requête comme ceçi :
```sql
SELECT r.id, SUM(t)
FROM recharge r, jsonb_path_query(price_ocpi, '$.elements.price_components[*].price') t
where id = 10
group by r.id
having SUM(t) > 0.0

```

# Avec Django
Lançons la requête depuis le code python :
`dictfetchall` et `namedtuplefetchall` sont deux fonctions, qui permettent récuperer le résultat du curseur ([voir ici](https://docs.djangoproject.com/fr/3.0/topics/db/sql/#executing-custom-sql-directly))
```python
from django.db import connection

def dictfetchall(cursor):
    "Return all rows from a cursor as a dict"
    columns = [col[0] for col in cursor.description]
    return [
        dict(zip(columns, row))
        for row in cursor.fetchall()
    ]


def namedtuplefetchall(cursor):
    "Return all rows from a cursor as a namedtuple"
    desc = cursor.description
    nt_result = namedtuple('Recharge', [col[0] for col in desc])
    return [nt_result(*row) for row in cursor.fetchall()]

with connection.cursor() as cursor:
    cursor.execute("""
                    SELECT r.id, SUM((elements #> '{price_components, 0}' ->> 'price')::float)
                    FROM recharge r , jsonb_array_elements(price_ocpi -> 'elements') elements
                    where r.id = 10
                    group by r.id
                    having SUM((mes_prices ->> 'price')::float) > 0.0
                    """)
    resultat = namedtuplefetchall(cursor)
```




# Conclusion

La gestion du json avec PostgreSQL et Django permet d'avoir un schéma flexible et de mélanger le meilleurs des mondes.

Cependant, si vous êtes réticents à l'utilisation de SQL, il vous faudra faire attention à la structure de votre JSON, car pour le moment
le requêtage avec l'ORM de Django est assez limité (pour le moment, je l'espère) (pour du **JSON**)


---
title: "Django prefetch_related et Prefetch"
date: 2020-08-08T10:45:33+02:00
draft: true
---


# _Introduction_
Dans cet article nous allons voir l'utilisation de la fonction `prefetch_related`, et de l'objet `Prefetch`. Comme spécifié dans la [doc](https://docs.djangoproject.com/fr/3.0/ref/models/querysets/#prefetch-related), cette fonction 
permet d'eviter un déluge de requête sur la base de donnée, bien utilisée elle permet des optimisations importantes, mais il faudra faire attention à certains
points, pour eviter d'être surpris par son fonctionnement.


#_Exemple_
Pour illustrer cela, on va reprendre l'exemple de la documentation :

```python
from django.db import models

class Ingredient(models.Model):
    nom = models.CharField(max_length=30)
    poids = models.FloatField()
    en_stock = models.Boolean()

class Pizza(models.Model):
    nom = models.CharField(max_length=50)
    ingredients = models.ManyToManyField(Ingredient)

    
```
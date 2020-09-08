---
title: "WIP Django prefetch_related et Prefetch"
date: 2020-09-07T10:45:33+02:00
draft: false
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
    en_stock = models.BooleanField()

    def __str__(self):
        return self.nom

class Pizza(models.Model):
    nom = models.CharField(max_length=50)
    ingredients = models.ManyToManyField(Ingredient)

    def __str__(self):
        return self.nom

    
```

Donc on va chercher tous nos ingredients, et on affiche les pizzas rattaché à l'ingredient :
````python
ingredients = Ingredient.objects.all()
for ingredient in ingredients:
     print(ingredient.pizza_set.all()
````

Voici les reqûetes SQL générées, ainsi que le print des pizzas. On peut voir que le code n'est pas optimisé
```
(0.004) SELECT "tuto_pizza"."id", "tuto_pizza"."nom" FROM "tuto_pizza" INNER JOIN "tuto_pizza_ingredients" ON ("tuto_pizza"."id" = "tuto_pizza_ingredients"."pizza_id") WHERE "tuto_pizza_ingredients"."ingredient_id" = 1 LIMIT 21; args=(1,)
<QuerySet [<Pizza: Régina>, <Pizza: Royale>]>
(0.001) SELECT "tuto_pizza"."id", "tuto_pizza"."nom" FROM "tuto_pizza" INNER JOIN "tuto_pizza_ingredients" ON ("tuto_pizza"."id" = "tuto_pizza_ingredients"."pizza_id") WHERE "tuto_pizza_ingredients"."ingredient_id" = 2 LIMIT 21; args=(2,)
<QuerySet [<Pizza: Régina>, <Pizza: Royale>]>
(0.001) SELECT "tuto_pizza"."id", "tuto_pizza"."nom" FROM "tuto_pizza" INNER JOIN "tuto_pizza_ingredients" ON ("tuto_pizza"."id" = "tuto_pizza_ingredients"."pizza_id") WHERE "tuto_pizza_ingredients"."ingredient_id" = 3 LIMIT 21; args=(3,)
<QuerySet [<Pizza: Régina>, <Pizza: Royale>]>
(0.001) SELECT "tuto_pizza"."id", "tuto_pizza"."nom" FROM "tuto_pizza" INNER JOIN "tuto_pizza_ingredients" ON ("tuto_pizza"."id" = "tuto_pizza_ingredients"."pizza_id") WHERE "tuto_pizza_ingredients"."ingredient_id" = 4 LIMIT 21; args=(4,)
<QuerySet [<Pizza: Régina>, <Pizza: Royale>, <Pizza: Chorizo>]>
(0.001) SELECT "tuto_pizza"."id", "tuto_pizza"."nom" FROM "tuto_pizza" INNER JOIN "tuto_pizza_ingredients" ON ("tuto_pizza"."id" = "tuto_pizza_ingredients"."pizza_id") WHERE "tuto_pizza_ingredients"."ingredient_id" = 5 LIMIT 21; args=(5,)
<QuerySet [<Pizza: Royale>]>
(0.001) SELECT "tuto_pizza"."id", "tuto_pizza"."nom" FROM "tuto_pizza" INNER JOIN "tuto_pizza_ingredients" ON ("tuto_pizza"."id" = "tuto_pizza_ingredients"."pizza_id") WHERE "tuto_pizza_ingredients"."ingredient_id" = 6 LIMIT 21; args=(6,)
<QuerySet [<Pizza: Royale>]>
(0.001) SELECT "tuto_pizza"."id", "tuto_pizza"."nom" FROM "tuto_pizza" INNER JOIN "tuto_pizza_ingredients" ON ("tuto_pizza"."id" = "tuto_pizza_ingredients"."pizza_id") WHERE "tuto_pizza_ingredients"."ingredient_id" = 7 LIMIT 21; args=(7,)
<QuerySet [<Pizza: Chorizo>]>
(0.001) SELECT "tuto_pizza"."id", "tuto_pizza"."nom" FROM "tuto_pizza" INNER JOIN "tuto_pizza_ingredients" ON ("tuto_pizza"."id" = "tuto_pizza_ingredients"."pizza_id") WHERE "tuto_pizza_ingredients"."ingredient_id" = 8 LIMIT 21; args=(8,)
<QuerySet [<Pizza: Chorizo>]>
```



Pour genérer moins de requête on va utiliser `prefetch_related` et precharger toutes les pizzas pour chaques ingredients.
Deux requêtes SQL sont générées et c'est tous.
```python
ingredients = Ingredient.objects.all().prefetch_related('pizza_set')
(0.001) SELECT "tuto_ingredient"."id", "tuto_ingredient"."nom", "tuto_ingredient"."poids", "tuto_ingredient"."en_stock" FROM "tuto_ingredient"; args=()
(0.002) SELECT ("tuto_pizza_ingredients"."ingredient_id") AS "_prefetch_related_val_ingredient_id", "tuto_pizza"."id", "tuto_pizza"."nom" FROM "tuto_pizza" INNER JOIN "tuto_pizza_ingredients" ON ("tuto_pizza"."id" = "tuto_pizza_ingredients"."pizza_id") WHERE "tuto_pizza_ingredients"."ingredient_id" IN (1, 2, 3, 4, 5, 6, 7, 8); args=(1, 2, 3, 4, 5, 6, 7, 8)
for ingredient in ingredients:
     print(ingredient.pizza_set.all())
   
<QuerySet [<Pizza: Régina>, <Pizza: Royale>]>
<QuerySet [<Pizza: Régina>, <Pizza: Royale>]>
<QuerySet [<Pizza: Régina>, <Pizza: Royale>]>
<QuerySet [<Pizza: Régina>, <Pizza: Royale>, <Pizza: Chorizo>]>
<QuerySet [<Pizza: Royale>]>
<QuerySet [<Pizza: Royale>]>
<QuerySet [<Pizza: Chorizo>]>
<QuerySet [<Pizza: Chorizo>]>

```

Cependant, pouvoir filtrer les pizzas, peut être utile, pour cela on va utiliser l'object `Prefetch`. (Il n'y que la pizza chorizo qui est en promotion)
````python
ingredients = Ingredient.objects.all().prefetch_related(Prefetch('pizza_set', queryset=Pizza.objects.filter(promotion=True)))
for ingredient in ingredients:
     print(ingredient, ingredient.pizza_set.all())

mozzarella <QuerySet []>
champignons de Paris <QuerySet []>
jambon <QuerySet []>
Coulis de tomate <QuerySet [<Pizza: Chorizo>]>
Oignons <QuerySet []>
lardons fumés <QuerySet []>
poivrons <QuerySet [<Pizza: Chorizo>]>
chorizo <QuerySet [<Pizza: Chorizo>]>
````

Bon imaginons que l'on a besoin de faire une annotation et de compter le nombre de pizza en promo par ingredient :

```python
ingredients = Ingredient.objects.all().prefetch_related(Prefetch('pizza_set', queryset=Pizza.objects.filter(promotion=True))).annotate(nb_promo=Count('pizza'))
for ingredient in ingredients:
     print(f"{ingredient}, {ingredient.pizza_set.all()}, {ingredient.nb_promo}")
```
> Voici le résultat (ci-dessous) et la il y a un probléme, notre  ``nb_promo`` ne correspond pas au notre de pizza dans notre queryset
```
Coulis de tomate, <QuerySet [<Pizza: Chorizo>]>, 3
lardons fumés, <QuerySet []>, 1
champignons de Paris, <QuerySet []>, 2
jambon, <QuerySet []>, 2
Oignons, <QuerySet []>, 1
poivrons, <QuerySet [<Pizza: Chorizo>]>, 1
mozzarella, <QuerySet []>, 2
chorizo, <QuerySet [<Pizza: Chorizo>]>, 1
```
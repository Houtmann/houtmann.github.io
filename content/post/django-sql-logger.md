---
title: "Logger les requêtes SQL sur son serveur Django (mode dev)"
date: 2020-08-01T10:45:33+02:00
draft: false
---

# _Introduction_
Je risque de mon repeter souvent sur ce blog, mais l'ORM de Django est très bien conçu.
On peut l'utiliser sans comprendre les base de données et le SQL, cependant, il est peut être traitre par moment (justement lorsque l'on comprend pas trop ce qui se passe derrière).
Je reviendrais dans mon blog avec plusieurs techniques d'optimisation (avec des cas concret ou non).

En attendant, il peut être utile lorsque l'on developpe sur son poste de voir les requêtes sql qui sont générées. Non pas pour les interpreter, mais pour voir, dans un premier temps le nombre de reqûetes qui sont générées


### Concretement

> à rajouter dans votre `settings.py`
```python
LOGGING = {
    'version': 1,
    'filters': {
        'require_debug_true': {
            '()': 'django.utils.log.RequireDebugTrue',
        }
    },
    'handlers': {
        'console': {
            'level': 'DEBUG',
            'filters': ['require_debug_true'],
            'class': 'logging.StreamHandler',
        }
    },
    'loggers': {
        'django.db.backends': {
            'level': 'DEBUG',
            'handlers': ['console'],
        }
    }
}
```

> Cela donne cela dans votre console de sortie.
```

DEBUG (0.002) SELECT "django_session"."session_key", "django_session"."session_data", "django_session"."expire_date" FROM "django_session" WHERE ("django_session"."expire_date" > '2020-08-02T15:53:23.488356+00:00'::timestamptz AND "django_session"."session_key" = '930xdns6qm56r79c1r7ee4b9z3dxk0lo'); args=(datetime.datetime(2020, 8, 2, 15, 53, 23, 488356, tzinfo=<UTC>), '930xdns6qm56r79c1r7ee4b9z3dxk0lo')
DEBUG (0.001) SELECT "auth_user"."id", "auth_user"."password", "auth_user"."last_login", "auth_user"."is_superuser", "auth_user"."username", "auth_user"."first_name", "auth_user"."last_name", "auth_user"."email", "auth_user"."is_staff", "auth_user"."is_active", "auth_user"."date_joined" FROM "auth_user" WHERE "auth_user"."id" = 2; args=(2,)
DEBUG (0.060) SELECT "django_admin_log"."id", "django_admin_log"."action_time", "django_admin_log"."user_id", "django_admin_log"."content_type_id", "django_admin_log"."object_id", "django_admin_log"."object_repr", "django_admin_log"."action_flag", "django_admin_log"."change_message", "auth_user"."id", "auth_user"."password", "auth_user"."last_login", "auth_user"."is_superuser", "auth_user"."username", "auth_user"."first_name", "auth_user"."last_name", "auth_user"."email", "auth_user"."is_staff", "auth_user"."is_active", "auth_user"."date_joined", "django_content_type"."id", "django_content_type"."app_label", "django_content_type"."model" FROM "django_admin_log" INNER JOIN "auth_user" ON ("django_admin_log"."user_id" = "auth_user"."id") LEFT OUTER JOIN "django_content_type" ON ("django_admin_log"."content_type_id" = "django_content_type"."id") WHERE "django_admin_log"."user_id" = 2 ORDER BY "django_admin_log"."action_time" DESC  LIMIT 10; args=(2,)
```

# Conclusion

En naviguant sur votre site en local, ou en consommant vos point d'api,vous serez peut être supris du nombre de requête genérer pour une seul vue.
Cela peut vous servir d'outils d'optimisation, car il permet rapidemment, d'identifier des parties de code qui pourrait être génératrice de nombreuses requête SQL
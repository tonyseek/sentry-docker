Sentry in Docker
==================

[![Requirements Status](https://requires.io/github/slafs/sentry-docker/requirements.png?branch=master)](https://requires.io/github/slafs/sentry-docker/requirements/?branch=master)

This is my approach for running [Sentry](https://getsentry.com) inside [Docker](https://docker.com/).
Almost everything here is configurable via environment variables (including DATABASES and CACHES settings).
It can be easily configured to run with redis (cache, buffers and celery broker), postgres database and LDAP authentication backend.

## Quickstart ##

to run a Sentry instance with default settings (with sqlite, locmem cache and no celery) run:

```
docker run -d --name=sentry --volume=/tmp/sentry:/data -p 80:9000 -e SECRET_KEY=randomvalue -e SENTRY_URL_PREFIX=http://sentry.mydomain.com slafs/sentry
```

You can visit now http://sentry.mydomain.com (assuming ``sentry.mydomain.com``
is mapped to your docker host) and login with default credentials
(username ``admin`` and password ``admin``) and create your first team and project.

Your sqlite database file and gunicorn logs are available in ``/tmp/sentry`` directory.

## Contributing ##

Try not to fork this repo just to create your own Docker image with some
minor tweak. Please open an [issue on GitHub](https://github.com/slafs/sentry-docker/issues)
and maybe we can include your use case directly within this image :).

You can even write a test case for your feature ;). See
[CONTRIBUTING.md](https://github.com/slafs/sentry-docker/blob/master/CONTRIBUTING.md).

Also feel free to give [feedback and comments](https://github.com/slafs/sentry-docker/issues)
about this image in general.

## Advanced usage ##

Copy the file with environment variables ``environment.example`` e.g ``cp environment.example environment``
and after tweaking some values run sentry like this

```
docker run -d --name=sentry --volume=/tmp/sentry:/data -p 80:9000 --env-file=environment slafs/sentry
```

``ENTRYPOINT`` for this image is a little wrapper script around default ``sentry`` executable.
Default ``CMD`` is ``start`` which runs ``upgrade`` command (initializes and upgrades the database)
and creates a default administrator (superuser) and then runs a http service (as in ``sentry --config=... start``).

All commands and args are passed to the ``sentry`` executable. This means

```
docker run [docker options] slafs/sentry --help
```

refers to running:

```
sentry --config=/conf/sentry.conf.py --help
```

inside the container.

### Admin user

You can specify a username, password and email address to create an initial sentry administrator.

Add those variables to your environment file

```
SENTRY_ADMIN_USERNAME=slafs
SENTRY_ADMIN_PASSWORD=mysecretpass
SENTRY_ADMIN_EMAIL=slafs@foo.com
```

See [django docs](https://docs.djangoproject.com/en/1.5/ref/django-admin/#createsuperuser) for details about ``createsuperuser`` command.

The default adminstrator username is ``admin`` with password ``admin`` (and ``root@localhost`` as email adress).

### Postgres

It is recommended to run your sentry instance with PostgreSQL database.
To link your sentry instance with a postgres container you can do it like this:

0. Pull an official PostgreSQL image: ``docker pull postgres`` (if you haven't already).
1. Run postgres container (from the official Docker image): ``docker run -d --name=postgres_container postgres``.
2. Add ``DATABASE_URL=postgres://postgres:@postgresdb/postgres`` to environment file.
3. Run sentry with linked postgres container: ```
docker run -d --name=sentry --volume=/tmp/sentry:/data -p 80:9000 --env-file=environment --link=postgres_container:postgresdb slafs/sentry```

Notice that an alias for your linked postgres container (``postgresdb``) is the same as a postgres host in ``DATABASE_URL`` variable.

``DATABASE_URL`` is a value that is parsed by an external app called [dj-database-url](https://github.com/kennethreitz/dj-database-url).


### Redis

Redis container can be used to improve sentry performance in number of ways. It can be used as a:

* cache backend,
* sentry buffers,
* celery broker.

You can link your sentry instance with a redis container like this:

0. Pull an official Redis image: ``docker pull redis`` (if you haven't already).
1. Run redis container (from the official Docker image): ``docker run -d --name=redis_container redis``
2. ```
docker run -d --name=sentry --volume=/tmp/sentry:/data -p 80:9000 --env-file=environment --link=postgres_container:postgresdb --link=redis_container:redis slafs/sentry```

If you want to use a different container alias for redis you should add ```SENTRY_REDIS_HOST=your_redis_alias``` to environment file.

#### Cache with redis

To use a redis cache backend add ``CACHE_URL=hiredis://redis:6379/2/``
to environment file (where ``redis`` is the alias of your linked redis container).
See [django-cache-url](https://github.com/ghickman/django-cache-url) docs for available formats.

#### Sentry buffers with redis

To use [sentry update buffers](http://sentry.readthedocs.org/en/latest/buffer/)
with redis you must add ``SENTRY_USE_REDIS_BUFFERS=True`` to environment file.

If you have many redis containers/hosts you can set a list of those hosts
in ``SENTRY_REDIS_BUFFERS`` variable so they can be used by sentry.
Like this: ``SENTRY_REDIS_BUFFERS=redis1:6380,redis2:6381``.

See [sentry docs](http://sentry.readthedocs.org/en/latest/buffer/#the-redis-backend) for details about redis buffer.

#### Celery with redis (and postgres)

To use Celery in sentry you must add ``CELERY_ALWAYS_EAGER=False`` to your environment file and run a celery worker like this:

```
docker run -d --name=sentry_celery_worker --link=redis_container:redis --link=postgres_container:postgresdb --volume=/tmp/sentry:/data --env-file=environment slafs/sentry celery worker -B
```

You can also set a different ``BROKER_URL`` via environment file by adding this:
``SENTRY_BROKER_URL=redis://otherredishost:6379/1``

You can run as many celery worker containers as you want but remember that only one of them should be run with ``-B`` option.

###Email

You can configure all [email settings](http://sentry.readthedocs.org/en/latest/quickstart/index.html#configure-outbound-mail)
by environment variables with ``SENTRY_`` prefix.
You have to also change an email backend and set it
to ``SENTRY_EMAIL_BACKEND=django.core.mail.backends.smtp.EmailBackend`` or something similar.

###LDAP
With this image You should be able to easily configure LDAP authentication for your Sentry instance.
To enable it add ``SENTRY_USE_LDAP=True`` to your ``environment`` file.
Then set the needed options by adding env variables with ``LDAP_``
prefix (see the table below). LDAP authentication backend is provided by
[django-auth-ldap](https://pythonhosted.org/django-auth-ldap/).

## Available environment variables

Refer to [sentry documentation](http://sentry.readthedocs.org/en/latest/config/index.html),
[django documentation](https://docs.djangoproject.com/en/1.5/ref/settings/)
and [django-auth-ldap documentation](https://pythonhosted.org/django-auth-ldap/reference.html)
for the meaning of each setting.

Environment variable name   | Django/Sentry setting                         | Type | Default value                                         | Description
----------------------------|-----------------------------------------------|------|-------------------------------------------------------|------------------------------------------------------------------------
SECRET_KEY                  | SECRET_KEY                                    |      | **REQUIRED!**                                         | set this to something random
SENTRY_URL_PREFIX           | SENTRY_URL_PREFIX                             |      | **REQUIRED!**                                         | no trailing slash!
DATABASE_URL                | DATABASES                                     |      | sqlite:////data/sentry.db                             |
CACHE_URL                   | CACHES                                        |      | locmem://                                             |
CELERY_ALWAYS_EAGER         | CELERY_ALWAYS_EAGER                           | bool | True                                                  |
SENTRY_BROKER_URL           | BROKER_URL                                    |      | ``redis://<SENTRY_REDIS_HOST>:<SENTRY_REDIS_PORT>/1`` |
SENTRY_REDIS_HOST           |                                               |      | redis                                                 |
SENTRY_REDIS_PORT           |                                               | int  | 6379                                                  |
SENTRY_WEB_HOST             | SENTRY_WEB_HOST                               |      | 0.0.0.0                                               |
SENTRY_WEB_PORT             | SENTRY_WEB_PORT                               | int  | 9000                                                  |
SENTRY_WORKERS              | SENTRY_WEB_OPTIONS['workers']                 | int  | 3                                                     | the number of gunicorn workers
SENTRY_USE_REDIS_BUFFER     |                                               | bool | False                                                 |
SENTRY_REDIS_BUFFERS        | SENTRY_REDIS_OPTIONS['hosts']*                | list | ``<SENTRY_REDIS_HOST>:<SENTRY_REDIS_PORT>``           | comma separated list of redis hosts (``host1:port1,host2:port2,...``)
SENTRY_EMAIL_BACKEND        | EMAIL_BACKEND                                 |      | django.core.mail.backends.console.EmailBackend        |
SENTRY_EMAIL_HOST           | EMAIL_HOST                                    |      | localhost                                             |
SENTRY_EMAIL_HOST_PASSWORD  | EMAIL_HOST_PASSWORD                           |      | ''                                                    |
SENTRY_EMAIL_HOST_USER      | EMAIL_HOST_USER                               |      | ''                                                    |
SENTRY_EMAIL_PORT           | EMAIL_PORT                                    | int  | 25                                                    |
SENTRY_EMAIL_USE_TLS        | EMAIL_USE_TLS                                 | bool | False                                                 |
SENTRY_SERVER_EMAIL         | SERVER_EMAIL                                  |      | root@localhost                                        |
SENTRY_ALLOW_REGISTRATION   | SENTRY_ALLOW_REGISTRATION                     | bool | False                                                 |
SENTRY_ADMIN_USERNAME       |                                               |      | admin                                                 | username for Sentry's superuser
SENTRY_ADMIN_PASSWORD       |                                               |      | admin                                                 | password for Sentry's superuser
SENTRY_ADMIN_EMAIL          |                                               |      | root@localhost                                        | email address for Sentry's superuser
SENTRY_DATA_DIR             |                                               |      | ``/data``                                             | custom location for logs and sqlite database
TWITTER_CONSUMER_KEY        | TWITTER_CONSUMER_KEY                          |      | ''                                                    |
TWITTER_CONSUMER_SECRET     | TWITTER_CONSUMER_SECRET                       |      | ''                                                    |
FACEBOOK_APP_ID             | FACEBOOK_APP_ID                               |      | ''                                                    |
FACEBOOK_API_SECRET         | FACEBOOK_API_SECRET                           |      | ''                                                    |
GOOGLE_OAUTH2_CLIENT_ID     | GOOGLE_OAUTH2_CLIENT_ID                       |      | ''                                                    |
GOOGLE_OAUTH2_CLIENT_SECRET | GOOGLE_OAUTH2_CLIENT_SECRET                   |      | ''                                                    |
GITHUB_APP_ID               | GITHUB_APP_ID                                 |      | ''                                                    |
GITHUB_API_SECRET           | GITHUB_API_SECRET                             |      | ''                                                    |
TRELLO_API_KEY              | TRELLO_API_KEY                                |      | ''                                                    |
TRELLO_API_SECRET           | TRELLO_API_SECRET                             |      | ''                                                    |
BITBUCKET_CONSUMER_KEY      | BITBUCKET_CONSUMER_KEY                        |      | ''                                                    |
BITBUCKET_CONSUMER_SECRET   | BITBUCKET_CONSUMER_SECRET                     |      | ''                                                    |
SENTRY_USE_LDAP             |                                               | bool | False                                                 | if set to ``False`` all other LDAP settings are discarded
LDAP_SERVER                 | AUTH_LDAP_SERVER_URI                          |      | ``ldap://localhost``                                  |
LDAP_BIND_DN                | AUTH_LDAP_BIND_DN                             |      | ''                                                    |
LDAP_BIND_PASSWORD          | AUTH_LDAP_BIND_PASSWORD                       |      | ''                                                    |
LDAP_USER_DN                | AUTH_LDAP_USER_SEARCH*                        |      | **REQUIRED!** if you want to use LDAP auth            | first argument of LDAPSearch (base_dn) when searching for users
LDAP_USER_FILTER            | AUTH_LDAP_USER_SEARCH*                        |      | ``(&(objectClass=inetOrgPerson)(cn=%(user)s))``       | third argument of LDAPSearch (filterstr) when searching for users
LDAP_GROUP_DN               | AUTH_LDAP_GROUP_SEARCH*                       |      | ''                                                    | first argument of LDAPSearch (base_dn) when searching for groups
LDAP_GROUP_FILTER           | AUTH_LDAP_GROUP_SEARCH*                       |      | ``(objectClass=groupOfUniqueNames)``                  | third argument of LDAPSearch (filterstr) when searching for groups
LDAP_GROUP_TYPE             | AUTH_LDAP_GROUP_TYPE*                         |      | ''                                                    | if set to 'groupOfUniqueNames' then ``AUTH_LDAP_GROUP_TYPE = GroupOfUniqueNamesType()``
LDAP_REQUIRE_GROUP          | AUTH_LDAP_REQUIRE_GROUP                       |      | None                                                  |
LDAP_DENY_GROUP             | AUTH_LDAP_DENY_GROUP                          |      | None                                                  |
LDAP_MAP_FIRST_NAME         | AUTH_LDAP_USER_ATTR_MAP['first_name']         |      | ``givenName``                                         |
LDAP_MAP_LAST_NAME          | AUTH_LDAP_USER_ATTR_MAP['last_name']          |      | ``sn``                                                |
LDAP_MAP_MAIL               | AUTH_LDAP_USER_ATTR_MAP['email']              |      | ``mail``                                              |
LDAP_GROUP_ACTIVE           | AUTH_LDAP_USER_FLAGS_BY_GROUP['is_active']    |      | ''                                                    |
LDAP_GROUP_STAFF            | AUTH_LDAP_USER_FLAGS_BY_GROUP['is_staff']     |      | ''                                                    |
LDAP_GROUP_SUPERUSER        | AUTH_LDAP_USER_FLAGS_BY_GROUP['is_superuser'] |      | ''                                                    |
LDAP_FIND_GROUP_PERMS       | AUTH_LDAP_FIND_GROUP_PERMS                    | bool | False                                                 |
LDAP_CACHE_GROUPS           | AUTH_LDAP_CACHE_GROUPS                        | bool | True                                                  |
LDAP_GROUP_CACHE_TIMEOUT    | AUTH_LDAP_GROUP_CACHE_TIMEOUT                 | int  | 3600                                                  |
LDAP_LOGLEVEL               |                                               |      | ``DEBUG``                                             | django_auth_ldap logger level (other values: NOTSET (to disable), INFO, WARNING, ERROR or CRITICAL)

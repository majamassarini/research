---
title: Distributed workers
authors: jpopelka
---

Currently, we run packit service in Openshift Online, because we need our API (httpd/service)
and dashboard to be publicly accessible. The most resources however consume workers,
which on the other hand don't need to be publicly available.
So the idea was to run (only) workers in an internal cluster which is not publicly accessible,
but we don't have to pay for it.

## Public broker & backend

Because service and workers communicate via Celery we need the broker (Redis)
and backend (PostgreSQL) to be publicly accessible.

### Route

If Redis/PostgreSQL were using HTTP(S) protocol, their exposing would be
as simple as creating a route for them.
Unfortunately (see [here](https://docs.openshift.com/container-platform/3.11/architecture/networking/routes.html#routers)
and [here](https://github.com/openshift/origin/issues/3415#issuecomment-137902453)),
routes are only for HTTP(S) or TLS+[SNI](https://en.wikipedia.org/wiki/Server_Name_Indication).

Both, [Redis (starting with latest version 6)](https://redis.io/topics/encryption)
and PostgreSQL can talk TLS, but they don't seem to have SNI support implemented, see
[here](https://github.com/redis/redis/issues/7210#issuecomment-626381723),
[here](https://www.postgresql.org/message-id/9af47a45-f92d-7ede-2d71-222ba24447eb%402ndquadrant.com)
and [here](https://www.postgresql.org/message-id/d05341b9-033f-d5fe-966e-889f5f9218e5%40proxel.se).

### Exposing a service

[Other options (than route) for exposing a service are Load Balancer, NodePort and ExternalIP](https://docs.openshift.com/container-platform/3.11/dev_guide/expose_service/index.html).
Unfortunately, none of them I succeeded to use on Openshift Online.

#### [Load Balancer](https://docs.openshift.com/container-platform/3.11/dev_guide/expose_service/expose_internal_ip_load_balancer.html)

Creating a Load Balancer is a privileged operation and we're not allowed to do that on OS Online.

#### [NodePort](https://docs.openshift.com/container-platform/3.11/dev_guide/expose_service/expose_internal_ip_nodeport.html)

Is a
[privileged operation](https://docs.openshift.com/container-platform/3.11/architecture/core_concepts/pods_and_services.html#service-nodeport)
and requires additional port resources.

#### [ExternalIP](https://docs.openshift.com/container-platform/3.11/dev_guide/expose_service/expose_internal_ip_service.html)

Downside of using ExternalIP
is that it [consumes an IP address](https://access.redhat.com/solutions/3178591), so you already need to have one to assign to it.

#### oc port-forward

Just for debugging/developing, needs to be done on client side.

## Other Celery supported brokers

Leaving aside that Redis can't do SNI, [quoting](https://redis.io/topics/security):
"Redis is designed to be accessed by trusted clients inside trusted environments.
This means that usually it is not a good idea to expose the Redis instance
directly to the internet or, in general, to an environment where untrusted clients
can directly access the Redis TCP port or UNIX socket."

### [RabbitMQ](https://docs.celeryproject.org/en/stable/getting-started/brokers/rabbitmq.html)

Supports [SSL/TLS](https://www.rabbitmq.com/ssl.html#erlang-ssl)
and [SNI as well](https://github.com/rabbitmq/rabbitmq-server/issues/789).

### [AWS SQS](https://docs.celeryproject.org/en/stable/getting-started/brokers/sqs.html)

[Price is \$0.5 per million messages](https://aws.amazon.com/sqs/pricing).
Currently we're producing/consuming around 1200 messages per day
so at this rate we'd be paying around 18 cents (\$0.018) a month for using SQS.

## Other SQLAlchemy/Celery supported databases

We use PostgreSQL via SQLAlchemy either [directly](https://github.com/packit-service/packit-service/blob/master/packit_service/models.py)
or as a [Celery result backend](https://github.com/packit-service/packit-service/blob/master/packit_service/celerizer.py#L49).
If we can't use it because it doesn't support SNI, then we have to find other free/open source,
[SQLAlchemy supported database](https://docs.sqlalchemy.org/en/13/dialects/index.html)
that supports TLS+SNI:

- SQLite - [nogo](https://github.com/packit-service/research/tree/master/data-stores#cons)
- [MariaDB](https://hub.docker.com/_/mariadb) - [supports TLS](https://mariadb.com/kb/en/securing-connections-for-client-and-server) but not [SNI](https://jira.mariadb.org/browse/MDEV-10658)
- [MySQL](https://hub.docker.com/_/mysql) - [supports TLS](https://dev.mysql.com/doc/refman/8.0/en/using-encrypted-connections.html) but not [SNI](https://bugs.mysql.com/bug.php?id=84849)
- [Firebird](https://hub.docker.com/r/jacobalberty/firebird/) - [supports TLS](https://www.firebirdsql.org/file/documentation/release_notes/html/en/3_0/rnfb30-security-new-authentication.html#rnfb30-security-ssltls), no reference to SNI

Or use cloud database, like [AWS RDS for PostgreSQL](https://aws.amazon.com/rds/postgresql)
whose [pricing](https://aws.amazon.com/rds/postgresql/pricing)
starts at $0.018/hour ($13/month) for [db.t3.micro](https://aws.amazon.com/rds/instance-types) (1GiB mem).

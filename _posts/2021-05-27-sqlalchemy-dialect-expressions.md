---
layout: post
title:  Dialect-specific Expressions in SQLAlchemy
date:   2021-05-27
categories: python
---

So it turns out SQLAlchemy doesn't really smooth over differences between
databases that much --- whoops! Maybe you, like me, need to write queries that
work across several engines (i.e., and so dialects) like SQLite, PostgreSQL,
etc.

Here's a *semi-nice* way I found to do that using the
[SQLAlchemy's compilation extension API](https://docs.sqlalchemy.org/en/14/core/compiler.html)
which avoids the hassle of having dialect-specific database classes, etc.

After defining this new expression element:

{% highlight python %}
from sqlalchemy.ext.compiler import compiles
from sqlalchemy.sql.expression import ColumnClause

class IfDialect(ColumnClause):
    def __init__(self, default, **dialect_elements):
        self.default_element = default
        self.dialect_elements = dialect_elements

@compiles(IfDialect)
def _compile_IfDialect(element, compiler, **kwargs):
    return compiler.process(
        element.dialect_elements.get(compiler.dialect.name, element.default_element),
        **kwargs
    )

if_dialect = IfDialect
{% endhighlight %}

You can switch out different expressions based on dialect, where the kwargs for
the expression match a particular dialect or default otherwise, like so:

{% highlight python %}
from sqlalchemy import column, func, select

statement = select(
    column("id"),
    if_dialect(
        default=func.to_char(column("dt"), "YYYY-MM-DD"),
        sqlite=func.strftime("%Y-%m-%d", column("dt")),
        mysql=func.date_format(column("dt"), "%Y-%m-%d"),
    ).label("dt_ymd")
)
{% endhighlight %}

And here's the output once compiled with those various dialects:

{% highlight python %}
from sqlalchemy.dialects import postgresql
from sqlalchemy.dialects import sqlite
from sqlalchemy.dialects import mysql

str(statement.compile(
    dialect=postgresql.dialect(), compile_kwargs={"literal_binds": "true"}))
# "SELECT id, to_char(dt, 'YYYY-MM-DD') AS dt_ymd"

str(statement.compile(
    dialect=sqlite.dialect(), compile_kwargs={"literal_binds": "true"}))
# "SELECT id, strftime('%Y-%m-%d', dt) AS dt_ymd"

str(statement.compile(
    dialect=mysql.dialect(), compile_kwargs={"literal_binds": "true"}))
# "SELECT id, date_format(dt, '%%Y-%%m-%%d') AS dt_ymd"
{% endhighlight %}

You can use it anywhere else you'd put a column. Here's a more
concrete example, which is what I needed it for:

{% highlight python %}
def _dt_statement(default, sqlite, mysql):
    dt_column = if_dialect(
        default=func.to_char(CoinValue.datetime, default),
        sqlite=func.strftime(sqlite, CoinValue.datetime),
        mysql=func.date_format(CoinValue.datetime, mysql)
    )

    return (
        select(CoinValue, func.max(CoinValue.datetime), dt_column)
        .group_by(CoinValue.coin_id, CoinValue, dt_column)
    )

hourly = _dt_statement(default="HH24", sqlite="%H", mysql="%H")
weekly = _dt_statement(default="YYYY-WW", sqlite="%Y-%W", mysql="%Y-%v")
daily = _dt_statement(default="YYYY-DDD", sqlite="%Y-%j", mysql="%Y-%j")
{% endhighlight %}

That's it. Enjoy!

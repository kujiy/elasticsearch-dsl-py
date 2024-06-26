.. _faceted_search:

Faceted Search
==============

The library comes with a simple abstraction aimed at helping you develop
faceted navigation for your data.

.. note::

    This API is experimental and will be subject to change. Any feedback is
    welcome.

Configuration
-------------

You can provide several configuration options (as class attributes) when
declaring a ``FacetedSearch`` subclass:

``index``
  the name of the index (as string) to search through, defaults to ``'_all'``.

``doc_types``
  list of ``Document`` subclasses or strings to be used, defaults to
  ``['_all']``.

``fields``
  list of fields on the document type to search through. The list will be
  passes to ``MultiMatch`` query so can contain boost values (``'title^5'``),
  defaults to ``['*']``.

``facets``
  dictionary of facets to display/filter on. The key is the name displayed and
  values should be instances of any ``Facet`` subclass, for example: ``{'tags':
  TermsFacet(field='tags')}``


Facets
~~~~~~

There are several different facets available:

``TermsFacet``
  provides an option to split documents into groups based on a value of a field, for example ``TermsFacet(field='category')``

``DateHistogramFacet``
  split documents into time intervals, example: ``DateHistogramFacet(field="published_date", calendar_interval="day")``

``HistogramFacet``
  similar to ``DateHistogramFacet`` but for numerical values: ``HistogramFacet(field="rating", interval=2)``

``RangeFacet``
  allows you to define your own ranges for a numerical fields:
  ``RangeFacet(field="comment_count", ranges=[("few", (None, 2)), ("lots", (2, None))])``

``NestedFacet``
  is just a simple facet that wraps another to provide access to nested documents:
  ``NestedFacet('variants', TermsFacet(field='variants.color'))``


By default facet results will only calculate document count, if you wish for
a different metric you can pass in any single value metric aggregation as the
``metric`` kwarg (``TermsFacet(field='tags', metric=A('max',
field=timestamp))``). When specifying ``metric`` the results will be, by
default, sorted in descending order by that metric. To change it to ascending
specify ``metric_sort="asc"`` and to just sort by document count use
``metric_sort=False``.

Advanced
~~~~~~~~

If you require any custom behavior or modifications simply override one or more
of the methods responsible for the class' functions:

``search(self)``
  is responsible for constructing the ``Search`` object used. Override this if
  you want to customize the search object (for example by adding a global
  filter for published articles only).

``query(self, search)``
  adds the query position of the search (if search input specified), by default
  using ``MultiField`` query. Override this if you want to modify the query type used.

``highlight(self, search)``
  defines the highlighting on the ``Search`` object and returns a new one.
  Default behavior is to highlight on all fields specified for search.


Usage
-----

The custom subclass can be instantiated empty to provide an empty search
(matching everything) or with ``query``, ``filters`` and ``sort``.

``query``
  is used to pass in the text of the query to be performed. If ``None`` is
  passed in (default) a ``MatchAll`` query will be used. For example ``'python
  web'``

``filters``
  is a dictionary containing all the facet filters that you wish to apply. Use
  the name of the facet (from ``.facets`` attribute) as the key and one of the
  possible values as value. For example ``{'tags': 'python'}``.

``sort``
  is a tuple or list of fields on which the results should be sorted. The format
  of the individual fields are to be the same as those passed to
  :meth:`~elasticsearch_dsl.Search.sort`.


Response
~~~~~~~~

the response returned from the ``FacetedSearch`` object (by calling
``.execute()``) is a subclass of the standard ``Response`` class that adds a
property called ``facets`` which contains a dictionary with lists of buckets -
each represented by a tuple of key, document count and a flag indicating
whether this value has been filtered on.

Example
-------

.. code:: python

    from datetime import date

    from elasticsearch_dsl import FacetedSearch, TermsFacet, DateHistogramFacet

    class BlogSearch(FacetedSearch):
        doc_types = [Article, ]
        # fields that should be searched
        fields = ['tags', 'title', 'body']

        facets = {
            # use bucket aggregations to define facets
            'tags': TermsFacet(field='tags'),
            'publishing_frequency': DateHistogramFacet(field='published_from', interval='month')
        }

        def search(self):
            # override methods to add custom pieces
            s = super().search()
            return s.filter('range', publish_from={'lte': 'now/h'})

    bs = BlogSearch('python web', {'publishing_frequency': date(2015, 6)})
    response = bs.execute()

    # access hits and other attributes as usual
    total = response.hits.total
    print('total hits', total.relation, total.value)
    for hit in response:
        print(hit.meta.score, hit.title)

    for (tag, count, selected) in response.facets.tags:
        print(tag, ' (SELECTED):' if selected else ':', count)

    for (month, count, selected) in response.facets.publishing_frequency:
        print(month.strftime('%B %Y'), ' (SELECTED):' if selected else ':', count)



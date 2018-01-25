esbulk
======

Fast parallel bulk loading utility for elasticsearch. Data is read from a
newline delimited JSON file or stdin and indexed into elasticsearch in bulk
*and* in parallel. The shortest command would be:

```shell
$ esbulk -index my-index-name < file.ldj
```

Caveat: If indexing *pressure* on the bulk API is too high (dozens or hundreds of
parallel workers, large batch sizes, depending on you setup), esbulk will halt
and report an error:

```shell
$ esbulk -index my-index-name -w 100 file.ldj
2017/01/02 16:25:25 error during bulk operation, try less workers (lower -w value) or
                    increase thread_pool.bulk.queue_size in your nodes
```

Please note that, in such a case, some documents are indexed and some are not.
Your index will be in an inconsistent state, since there is no transactional
bracket around the indexing process.

However, using defaults (parallism: number of cores) on a single node setup
will just work. For larger clusters, increase the number of workers until you
see full CPU utilization. After that, more workers won't buy any more speed.

Installation
------------

    $ go get github.com/miku/esbulk/cmd/esbulk

For `deb` or `rpm` packages, see: https://github.com/miku/esbulk/releases

Usage
-----

    $ esbulk -h
    Usage of esbulk -index NAME [OPTIONS] FILE:
      -cpuprofile string
          write cpu profile to file
      -host string
          elasticsearch host (default "localhost")
      -id string
          name of field to use as id field, by default ids are autogenerated
      -index string
          index name
      -mapping string
          mapping string or filename to apply before indexing
      -memprofile string
          write heap profile to file
      -port int
          elasticsearch port (default 9200)
      -purge
          purge any existing index before indexing
      -server string
          elasticsearch server, this works with https as well (default "http://localhost:9200")
      -size int
          bulk batch size (default 1000)
      -type string
          elasticsearch doc type (default "default")
      -u string
          http basic auth username:password, like curl -u
      -v  prints current program version
      -verbose
          output basic progress
      -w int
          number of workers to use (default 4)
      -z  unzip gz'd file on the fly

![](https://raw.githubusercontent.com/miku/esbulk/master/docs/asciicast.gif)

To index a JSON file, that contains one document
per line, just run:

    $ esbulk -index example file.ldj

Where `file.ldj` is line delimited JSON, like:

    {"name": "esbulk", "version": "0.2.4"}
    {"name": "estab", "version": "0.1.3"}
    ...

By default `esbulk` will use as many parallel
workers, as there are cores. To tweak the indexing
process, adjust the `-size` and `-w` parameters.

You can index from gzipped files as well, using
the `-z` flag:

    $ esbulk -z -index example file.ldj.gz

Starting with 0.3.7 the preferred method to set a
non-default server hostport is via `-server`, e.g.

    $ esbulk -server https://0.0.0.0:9201

This way, you can use https as well, which was not
possible before. Options `-host` and `-port` are
kept for backwards compatibility.

Reusing IDs
-----------

Since version 0.3.8: If you want to reuse IDs from your documents in elasticsearch, you
can specify the ID field via `-id` flag:

    $ cat file.json
    {"x": "doc-1", "db": "mysql"}
    {"x": "doc-2", "db": "mongo"}

Here, we would like to reuse the ID from field *x*.

    $ esbulk -id x -index throwaway -verbose file.json
    ...

    $ curl -s http://localhost:9200/throwaway/_search | jq
    {
      "took": 2,
      "timed_out": false,
      "_shards": {
        "total": 5,
        "successful": 5,
        "failed": 0
      },
      "hits": {
        "total": 2,
        "max_score": 1,
        "hits": [
          {
            "_index": "throwaway",
            "_type": "default",
            "_id": "doc-2",
            "_score": 1,
            "_source": {
              "x": "doc-2",
              "db": "mongo"
            }
          },
          {
            "_index": "throwaway",
            "_type": "default",
            "_id": "doc-1",
            "_score": 1,
            "_source": {
              "x": "doc-1",
              "db": "mysql"
            }
          }
        ]
      }
    }

Nested ID fields
----------------

Version 0.4.3 adds support for nested ID fields:

```
$ cat fixtures/pr-8-1.json
{"a": {"b": 1}}
{"a": {"b": 2}}
{"a": {"b": 3}}

$ esbulk -index throwaway -id a.b < fixtures/pr-8-1.json
...
```

Concatenated ID
---------------

Version 0.4.3 adds support for IDs that are the concatenation of multiple fields:

```
$ cat fixtures/pr-8-2.json
{"a": {"b": 1}, "c": "a"}
{"a": {"b": 2}, "c": "b"}
{"a": {"b": 3}, "c": "c"}

$ esbulk -index throwaway -id a.b,c < fixtures/pr-8-1.json
...

      {
        "_index": "xxx",
        "_type": "default",
        "_id": "1a",
        "_score": 1,
        "_source": {
          "a": {
            "b": 1
          },
          "c": "a"
        }
      },
```

Using X-Pack
------------

Since 0.4.2: support for secured elasticsearch nodes:

```
$ esbulk -u elastic:changeme -index myindex file.ldj
```

----

A similar project has been started for solr, called [solrbulk](https://github.com/miku/solrbulk).

Contributors
------------

* [klaubert](https://github.com/klaubert)
* [sakshambathla](https://github.com/sakshambathla)


Measurements
------------

```shell
$ csvlook -I measurements.csv
| es_version | esbulk_version | docs      | average_doc_size_b | machines | cores | heap_gb | duration_s | doc_per_s |
| ---------- | -------------- | --------- | ------------------ | -------- | ----- | ------- | ---------- | --------- |
| 6.1.2      | 0.4.8          | 138000000 | 2000               | 1        | 32    | 64      | 6420       | 22100     |

```

Why not add a [row](https://github.com/miku/esbulk/pulls)?

# ftm-startkit

**Got questions or improvements? Please file a pull request!**

This is an onboarding guide for data journalists and developers for working on projects using the *FollowTheMoney* toolkit.

It is based on the work of the [Organized Crime and Corruption Reporting Project (OCCRP)](https://occrp.org) and [OpenSanctions](https://opensanctions.org) but strongly opinionated on how [investigativedata.io](https://investigativedata.io)-projects are using and extending this patterns and tool stack.

Our platforms like [farmsubsidy.org](https://farmsubsidy.org), [opensecuritydata.eu](https://opensecuritydata.eu), [followthegrant.org](https://followthegrant.org), [correctiv.org/spendengerichte](https://correctiv.org/spendengerichte) (and more to come) are all sharing the same data model and conceptual patterns (which are described in this document). This allows us to efficiently build projects and databases like the mentioned ones using shared libraries for the **data model**, **deduplication**, **transformation**, but as well **data warehousing**, a **query dsl** which enables a rich **api** that can feed a **frontend**.

All of the technology used in this stack is under active development and *work in progress*, as we do use each project to improve the underlying shared libraries.

The main programming language for this stack is `Python` 3.11

## data model

The basis is `followthemoney`, which is a python library containing the actual data model via a `yaml` specification and the implementation to work with this data within python applications.

FollowTheMoney allows *real world data* used in investigations to map to a common data model that describes **Entities** such as *Persons*, *Companies* and *Organizations*, and the *Relations* between them. Each **Entity** has a `schema` and a set of properties this type of entity can have. A person for example has properties for a `name` or the `birthDate`. All properties are always *multi valued* (A person can have multiple names and even birth dates, which is unfortunately common in *arbitrary data from the wild*). **Entities** have a unique ID that allows us to reference them within other entities (e.g., a `Directorship` describing the relation between a `Person` and a `Company`).

Entities are often expressed as snippets of `JSON`, with three standard fields: a unique `id`, a specification of the type of the entity called `schema`, and a set of `properties`. `properties` are multi-valued and values are always strings.

```json
{
  "id": "1b38214f88d139897bbd13eabde464043d84bbf9",
  "schema": "Person",
  "properties": {
    "name": ["Jane Doe"],
    "nationality": ["us", "au"],
    "birthDate": ["1982"]
  }
}
```

### followthemoney resources

[followthemoney data model explorer](https://followthemoney.tech/explorer/)

[followthemoney python docs](https://followthemoney.tech/docs/)

[followthemoney github](https://github.com/alephdata/followthemoney)

[followthemoney-store](https://github.com/alephdata/followthemoney-store) - a SQL storage implementation for follow the money data

### Dataset and DataCatalog

Within `followthemoney`, there is no information about what **Dataset** an entity belongs to. That's why [OpenSanctions](https://opensanctions.org) introduced this concept in their own base library `nomenklatura`. It extends the followthemoney logic by introducing a concept for a `Dataset` which can belong to a `DataCatalog`. Entities within `nomenklatura` can actually belong to more than one `Dataset` and are therefore called (as in the python class) a `CompositeEntity` (as they are generated or merged from fragmented information from different data sources.)

### Statements

Another fundamental extension by `nomenklatura` is the ability to describe entities as a list of `Statements`. Therefore, the data format doesn't look anymore like the `json` example described above, but rather `csv`'ish as each data record describes one property and it's value for an entity (simplified):

```csv
id,dataset,schema,prop,value
1,my_dataset,Person,firstName,Jane
1,my_dataset,Person,lastName,Doe
1,my_dataset,Person,nationality,us
1,my_dataset,Person,nationality,au
```

`nomenklatura` provides functionality to turn entities into their statements and vice versa.

[nomenklatura on github](https://github.com/opensanctions/nomenklatura)

## data storage

*FollowTheMoney* data can be stored in line-based `json` files. This is the main data format to provide datasets, share them between projects or other data stores.

A convention is to store follow the money data in line-based json files named `entities.ftm.json`

As the data model is as well used in [Aleph](https://aleph.occrp.org), data can be exported from or imported to any aleph instance using the `alephclient`.

Other data stores are implemented in `nomenklatura`, and extended in `ftmq` to provide more data querying possibilities. The `ftmq` library provides helpers to import and export datasets from `json` into a **SQL database**, **LevelDB** or even [Clickhouse](https://clickhouse.com/).

### ftmq

[ftmq](https://github.com/investigativedata/ftmq)

`ftmq` is our own extension of the logic provided by `followthemoney` and `nomenklatura`. It contains a lot of helpers for reading and writing arbitrary data sources such as local files, remote http(s) resources or protocols like s3, ftps, and many more (based on [fsspec](https://filesystem-spec.readthedocs.io/en/latest/usage.html))

It uses the data stores introduced by `nomenklatura` but extends them with an extensive **query language** to ask specific questions to a *FollowTheMoney* dataset, for example "filter all the `Payments` for 2023 that are higher than €10.000".

## generating datasets

The base library `followthemoney` provides a [mapping interface to create entities from structured data sources](https://followthemoney.tech/docs/mappings/).

But to handle more use cases and write less (repeated) python code for each dataset, we extended this functionality with [investigraph](https://investigraph.dev).

**investigraph** is an [ETL](https://en.wikipedia.org/wiki/Extract,_transform,_load) framework that allows research teams to build their own data catalog themselves as easily and reproducable as possible. The **investigraph** framework provides logic for *extracting*, *transforming* and *loading* any data source into [followthemoney entities](https://followthemoney.tech/).

For most common data source formats, this process is possible without programming knowledge, by means of an easy `yaml` specification interface. However, if it turns out that a specific dataset can not be parsed with the built-in logic, a developer can plug in *custom python scripts* at specific places within the pipeline to fulfill even the most edge cases in data processing.

**investigraph** is our main framework to deal with followthemoney data generation.

[investigraph documentation](https://docs.investigraph.dev)

[investigraph tutorial](https://docs.investigraph.dev/tutorial)

[Example dataset scrapers based on investigraph](https://github.com/investigativedata/investigraph-eu)

Scraping code for datasets can live in different git repositories and combined into one (or more) data catalogs.

`investigraph` is built on top of [prefect.io](https://prefect.io) and is supported by the [Media Tech Lab Bayern](https://github.com/media-tech-lab)

## deduplication

One of the core tasks when working on a followthemoney data project is deduplicating similar entities and cross-referencing entities from different datasets. There are several ways to do that:

[Cross-referencing in Aleph](https://docs.aleph.occrp.org/users/investigations/cross-referencing/)

This creates *Profiles* of merged entities within Aleph.

But, more programatically, deduplication can achived using `nomenklatura`. [README](https://github.com/opensanctions/nomenklatura). This library provides a simple UI client to manually check deduplication candidates as well as a scoring algorithm to automatically merge similar entities based on a score threshold.

`nomenklatura` generates *resolver files* that describe the matching decisions of entities based on a score. This allows reproducible data workflows in the sense of:

1. build a dataset from scratch (e.g. with `investigraph`)
2. apply already decided matching to it via the *resolver file*
3. manually continue deduplication via `nomeklatura`
4. repeat

We are currently working on an exchange format and client for turning *Aleph profiles* into *nomenklatura resolver files* and vice versa.

## api

Once we have one or more clean and deduplicated datasets, we probably want to show it to the public. That's when `ftmstore-fastapi` comes in, a python application that is built on top of [FastAPI](https://fastapi.tiangolo.com/) that can read any store implemented in `nomenklatura` + `ftmq` (as in SQL, LevelDB, Clickhouse) and expose it as an api to allow filtering, sorting and searching.

[Example api with its query documentation](https://api.investigraph.dev/)

[ftmstore-fastapi on github](https://github.com/investigativedata/ftmstore-fastapi)

## frontend

Once we have our api ready, we want to build a nice data exploration frontend. [An example lives here](https://investigraph.dev/datasets).

The frontend application uses [ftm-joy-ui](https://github.com/investigativedata/ftm-joy-ui/) to provide `React` components based on Material [Joy UI](https://mui.com/joy-ui/getting-started/) and some api fetching logic to use for an individual app. These apps are usually built with [Next.js](https://nextjs.org/), but the helper library `ftm-joy-ui` would work for other `React` frameworks, too.

[Example Next.js followthemoney data explorer on github](https://github.com/investigativedata/investigraph-site)

## requirements

These python packages are needed to work with the described stack:

```
followthemoney
nomenklatura
ftmq
investigraph
alephclient
```

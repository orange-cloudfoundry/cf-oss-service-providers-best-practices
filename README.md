# cf-oss-service-providers-best-practices

Collects best practices from opensource cloudfoundry platforms service providers, potentially including 
* backup/restores
* monitoring
* log aggregation
* ssh jumpbox


# FAQ

## What documentation is suitable for this repo ?

By default, documentation and best practices that applies to the core of CF managed by the CFF foundation should instead be contributed to the [cloudfoundry/docs-*](https://github.com/cloudfoundry?utf8=%E2%9C%93&q=docs&type=&language=) repositories (the source of http://docs.cloudfoundry.org). The core of CF is any feature from repos hosted on https://github.com/cloudfoundry or https://github.com/cloudfoundry-incubator

This repo collects CF-related analysis and best practices that don't fit well with:
* existing official [cloudfoundry/docs-*](https://github.com/cloudfoundry?utf8=%E2%9C%93&q=docs&type=&language=) repositories, 
* or documentation of individual opensource projects (such as https://github.com/cloudfoundry-community/ or https://github.com/orange-cloudfoundry), e.g. because content spans many projects

This may include strategies/comparative analysis for features that fall outside the scope of official CFF distribution:
- would typically be in the differentiation scope domains for certified CF distributions or opensource community contributions, 
- would be too specific to certain types of deployment/use-case (altough we try to avoid Orange-specific use-cases here).


## How to contribute images/diagrams

Please contribute source of the diagram as well as a rendered form (JPG or PNG) to enable future improvements without a large extra work.
We strive to ease contributions without requiring specific licenses or operating system. 

For images where you need precise control over the layout, consider SaaS drawing tools with free tiers and source exports, such as draw.io (see inspiration in [prometheus-boshrelease docs](https://github.com/cloudfoundry-community/prometheus-boshrelease/tree/3c93dd8531706344a0a9d53ffca972fae3e61ff0/docs)

For images where no tight control over layout is required, consider [plantuml](http://plantuml.com/) syntax, possibly pointing to remotely rendered PNG/JPG from http://plantuml.com/plantuml/ 

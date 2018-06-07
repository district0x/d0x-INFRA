# Creating District

Following document goes through usual project structure used for a district codebase. In future we'll provide lein template to generate suggested project structure. 

## Server
Main purpose of server side code is to crawl application data from blockchain and IPFS and restructure it into regular database, so users can perform searchings & filterings over the data. 
Files for server side are located at `/src/<project-name>/server/`.

#### dev.cljs
Contains configuration for running server in dev mode by starting [mount](https://github.com/tolitius/mount). Also, contains function `redeploy` that redeploys smart contracts and restarts all related mount modules. 

#### core.cljs
Contains configuration for running server in prod mode. 

#### deployer.cljs 
Contains mount module, that deploys and initializes application smart contracts. 

#### syncer.cljs
Mount module that listens to application related events on blockchain and inserts data into traditional database. 

#### db.cljs
Contains mount module to setup a traditional database and provide functions for simpler operating. Currently our preferred DB is SQLite, for which we use wrapper module [district-server-db](https://github.com/district0x/district-server-db). Therefore the main purpose of db.cljs is to create SQL tables at mount start. 

#### generator.cljs
Contains mount module that generates blockchain transactions in order to initialize application with mock data. 

#### contract/
We use contract folder for defining namespaces that reflect API of smart contracts. For example `contract/my-great-contract.cljs` will contain functions for calling methods in MyGreatContract.sol.

#### emailer.cljs
Contains mount module that listens to blockchain events and sends out notification emails accordingly. 

#### Recommended district server modules:
Here's list of [mount](https://github.com/tolitius/mount) modules we usually use to reuse same logic across districts.

- [district-server-config](https://github.com/district0x/district-server-config) - Reads configuration from file on mount start.
- [district-server-db](https://github.com/district0x/district-server-db) - Sets up SQLite DB and provides helper functions. 
- [district-server-graphql](https://github.com/district0x/district-server-graphql) - Sets up GraphQL server.t
- [district-server-smart-contracts](https://github.com/district0x/district-server-smart-contracts) - Keeps track of smart-contract addresses and provides helper functions.
- [district-server-web3](https://github.com/district0x/district-server-web3) - Provides Web3 instance.
- [district-server-web3-watcher](https://github.com/district0x/district-server-web3-watcher) - Watches connection to Ethereum blockchain.
- [district-server-logging](https://github.com/district0x/district-server-logging) - Sets up server logging.

Here's full list of [district-server-modules](https://github.com/search?q=topic%3Adistrict-server-module+org%3Adistrict0x&type=Repositories).

## UI
UI part of a district is made using [re-mount](https://github.com/district0x/d0x-INFRA/blob/master/re-mount.md) modularisation pattern. 
Files for front-end side are located at `/src/<project-name>/ui/`.

#### core.cljs
Includes all re-mount modules and starts mount. 

#### Recommended file structure for UI codebase
Each page or logical module has own folder inside `/src/<project-name>/ui/` and contains one or more of following files: 

_Note, there isn't difference if the folder represents [re-mount](https://github.com/district0x/d0x-INFRA/blob/master/re-mount.md) module, page or just some set of events and subscriptions._

#### page.cljs
Defines [multimethod page](https://github.com/district0x/district-ui-router#districtuicomponentpage) + subcomponents. 

#### events.cljs
re-frame events related to the page or module. 

#### subs.cljs
re-frame subscription related to the page or module. 

#### core.cljs
Used when folder is [re-mount](https://github.com/district0x/d0x-INFRA/blob/master/re-mount.md) module to define module with `defstate`

#### component/
Reagent components provided by the module or used on the page. 


#### Recommended district UI modules:
- [district-ui-graphql](https://github.com/district0x/district-ui-graphql) - Client-side solution for GraphQL.
- [district-ui-web3](https://github.com/district0x/district-ui-web3) - Provides Web3 Instance
- [district-ui-router](https://github.com/district0x/district-ui-router) - Provides routing functionality.
- [district-ui-smart-contracts](https://github.com/district0x/district-ui-smart-contracts) - Loads smart contract files.
- [district-ui-web3-accounts](https://github.com/district0x/district-ui-web3-accounts) - Loads web3 accounts. 
- [district-ui-web3-balances](https://github.com/district0x/district-ui-web3-balances) - Loads web3 balances.
- [district-ui-web3-tx](https://github.com/district0x/district-ui-web3-tx) - Manages web3 transactions. 
- [district-ui-web3-tx-log](https://github.com/district0x/district-ui-web3-tx-log) - Provides Transaction Log functionality. 
- [district-ui-web3](https://github.com/district0x/district-ui-web3) - Provides Web3 instance


Here's full list of [district-ui-modules](https://github.com/search?q=topic%3Adistrict-ui-module+org%3Adistrict0x&type=Repositories).

## Shared
Contains all code that's supposed to be shared between server and UI. 
Files for are located at `/src/<project-name>/shared/`.

#### smart-contracts.cljs
Contains map defining smart contracts addresses. This file is usually edited programatically when deployer deploys contracts and writes new addresses into it. 

#### routes.cljs 
Definition of routes for UI. It's in shared namespace, because server usually also needs to form URLs when sending email notifications. 

#### graphql_schema.cljs
Contains definition for GraphQL schema. 

#### contract/
Contract folder contains namespaces providing parser functions when loading data from blockchain contract. 

## Styles
For visual styling we use [garden](https://github.com/noprompt/garden) with files being located at `/src/<project-name>/styles/`. Our recommended way of naming classes is to use noun-like definitions, that semantically express what the component is, not how it looks. For example:

```
[:div.meme-card]
[:div.nav-link.active]
```
Here we have 'div.meme-card' as a single "type" identificator, with optional status class. A necessary and sufficient to be identified in CSS. This should prevent styling concerns leaking into the code and avoiding "clash of classes" in responsive design.

Example of adjective-like classes to avoid:

```clojure
;; this is not recommended naming, it's expressing how component looks
[:div.meme.padded.small.elevated]
```
Adjective-like "classes" are perfectly good as a class mixins in the styles layer of the applications:
```clojure
[:.meme
  (merge 
    (box-rounded 5)
    {:background "color"})]
```

## Smart Contracts
Solidity smart contract files are located at `resources/public/contracts/src` and by using [lein-solc](https://github.com/district0x/lein-solc) they're usually compiled into `resources/public/contracts/build`. 

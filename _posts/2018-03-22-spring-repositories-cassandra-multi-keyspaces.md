---
layout: post
title:  "Spring Data Cassandra: Connect To Multiple Keyspaces Via Reactive Repositories"
date:   2018-04-13
excerpt: ""
keywords:
  - spring
  - spring data
  - repository
  - cassandra
  - apache cassandra
  - java
  - kotlin
  - cybersearch
---

Recently, **Spring 5.0** and **Spring Boot 2.0** were released. My team heavily using both for a while. I personally love **Spring**
 ecosystem moving to reactive paradigm, beside **Kotlin** support. 
 
For some services we run **Apache Cassandra**  as backing storage. Initially, we use raw [Datastax Java driver]() to communicate with db. 
 Its a solid driver with non-blocking api exposed through Guava's `ListenableFuture`. Despite driver is cool, it provides too low level 
 interfaces. At some point, we found us writing similar to **Spring Data Repository** wrappers around default api. So, we decided 
 not to reinvent wheels and try **Spring Data Cassandra**.
 
The first issue we faced using that library was about connecting to multiple keyspaces within single repositories 
 context. Lack of official documentation on this theme forced us to read source code and found out solution. To 
 demonstrate it, lets start by defining our cql model(I will skip entities details).
 
``` sql
CREATE TABLE IF NOT EXISTS bitcoin.block
CREATE TABLE IF NOT EXISTS bitcoin.tx
CREATE TABLE IF NOT EXISTS ethereum.block
CREATE TABLE IF NOT EXISTS ethereum.tx
``` 

Than, we process by creating entities. One important note here: we should use **separate** packages for 
 keyspaces we want to use in one context.
``` kotlin
package cybersearch.cassandra.bitcoin.model

import org.springframework.data.cassandra.core.mapping.Table

@Table("block") data class CqlBitcoinBlock
@Table("tx") data class CqlBitcoinTx
```
``` kotlin
package cybersearch.cassandra.ethereum.model

import org.springframework.data.cassandra.core.mapping.Table

@Table("block") data class CqlEthereumBlock
@Table("tx") data class CqlEthereumTx
```

So, at this point we have entities definitions. Move on and create spring reactive repositories.
``` kotlin
package cybersearch.cassandra.bitcoin.repository

interface BitcoinBlockRepository : ReactiveCassandraRepository<CqlBitcoinBlock, String>
interface BitcoinTxRepository : ReactiveCassandraRepository<CqlBitcoinTx, String>
```
``` kotlin
package cybersearch.cassandra.ethereum.repository

interface EthereumBlockRepository : ReactiveCassandraRepository<CqlEthereumBlock, String>
interface EthereumTxRepository : ReactiveCassandraRepository<CqlEthereumTx, String>
```

Here we come to the most lovely spring part: configuration part. Tons of magic, a lot of beens and everything else to work 
 our repositories. As usual we should provide Spring Data configuration by extending base configuration class:

``` kotlin
package cybersearch.cassandra.bitcoin.configuration

@Configuration
@EnableReactiveCassandraRepositories(
        basePackages = ["cybersearch.cassandra.bitcoin.repository"],
        reactiveCassandraTemplateRef = "bitcoinKeyspaceCassandraTemplate"
)
class BitcoinRepositoriesConfiguration : AbstractReactiveCassandraConfiguration {

    override fun getKeyspaceName(): String = "bitcoin"
    override fun getEntityBasePackages(): Array<String> = arrayOf("cybersearch.cassandra.bitcoin.model")

    @Bean("bitcoinKeyspaceCassandraTemplate")
    fun reactiveCassandraTemplate(): ReactiveCassandraOperations {
        return ReactiveCassandraTemplate(
            DefaultReactiveSessionFactory(reactiveSession()), cassandraConverter()
        )
    }

    @Bean("bitcoinKeyspaceReactiveSession")
    fun reactiveSession(): ReactiveSession = DefaultBridgedReactiveSession(session().`object`)

    @Bean("bitcoinKeyspaceSession")
    override fun session(): CassandraSessionFactoryBean {
        val session = super.session()
        session.setKeyspaceName(keyspaceName)
        return session
    }
}
```

For Ethereum, create same logical configuration. Now, we are done. Enjoy reactive repositories!
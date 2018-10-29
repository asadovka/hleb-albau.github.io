---
layout: post
title:  "Gradle + Kotlin JUnit5 + Jacoco + Coverall.io Integration"
date:   2018-06-14
excerpt: ""
keywords:
  - code coverage
  - java
  - kotlin
  - coverall
  - coverall.io
  - jacoco
  - gradle
  - junit
  - junit5
  - junit-jupiter
  - multiproject
---

 Testing is very powerful technique used in software development for years. To check all conditions/branches/lines is 
covered via automatic tests engineers use various test coverage utils/libs. Today I want to show how to enable great
service **Coverall.io** to track test coverage for your **Kotlin** + **Gradle** opensource project.

## Regular gradle stuff
Starting regular gradle stuff with buildscript, repositories and wrapper task:

```gradle
repositories {
    jcenter()
}

buildscript {

    repositories {
        jcenter()
    }
    
    dependencies {
        classpath("org.jetbrains.kotlin:kotlin-gradle-plugin:1.2.50")
        classpath("org.kt3k.gradle.plugin:coveralls-gradle-plugin:2.8.2")
    }
}

task wrapper(type: Wrapper) {
    gradleVersion = '4.8'
    distributionType = Wrapper.DistributionType.ALL
}
```


## Jacoco

 To collect coverage metrics we will use [JaCoCo tool](https://www.eclemma.org/jacoco/). Jacoco has two tasks. The first
used to generate file containing processing results with internal **.exec** format. Next one, **JacocoReport** 
library task converts **.exec** file to various *view* representations (xml, csv, html and etc). Suddenly, Jacoco by
default calculate coverage for each module separately. To calculate overall project test coverage we should:

* Generate **.exec** file for each individual module    
* Merge **.exec** files from all submodules into single file (We will create **jacocoMergeSubprojectResultsIntoRootOne** task)
* Using merged **.exec** file generate required report views (We will create **jacocoRootReport** task)

Let's start by applying Jacoco plugin for all projects. 

```gradle
allprojects {

    apply plugin: "jacoco"

    jacoco {
        toolVersion = "0.8.2"
    }
}
```

Then, register Jacoco in subprojects JUnit5 test scope to generate **$buildDir/jacoco/moduleTestsCoverage.exec** files.
```gradle
subprojects {

    apply plugin: "kotlin"
    
    dependencies {
        compile("org.junit.jupiter:junit-jupiter-api:5.2.0")
        compile("org.junit.jupiter:junit-jupiter-engine:5.2.0")
    }
    
    test {
        useJUnitPlatform()
        jacoco {
            append = false
            destinationFile = file("$buildDir/jacoco/moduleTestsCoverage.exec")
            includeNoLocationClasses = true
            excludes = ['jdk.internal.*']
        }
    }
}
```

And finally, define two previously mentioned tasks:
```gradle
def allTestsCoverageFile = "$buildDir/jacoco/rootTestsCoverage.exec"

task jacocoMergeSubprojectResultsIntoRootOne(type: JacocoMerge) {
    destinationFile = file(allTestsCoverageFile)
    executionData = project.fileTree(dir: '.', include: '**/build/jacoco/moduleTestsCoverage.exec')
}

task jacocoMerge(dependsOn: ['jacocoMergeSubprojectResultsIntoRootOne'])

task jacocoRootReport(type: JacocoReport, dependsOn: "jacocoMerge") {
    reports {
        xml.enabled = true
        html.enabled = true
        xml.destination file("${buildDir}/reports/jacoco/test/jacocoTestReport.xml")
    }
    additionalSourceDirs = files(subprojects.sourceSets.main.allSource.srcDirs)
    sourceDirectories = files(subprojects.sourceSets.main.allSource.srcDirs)
    classDirectories = files(subprojects.sourceSets.main.output)
    executionData = files(allTestCoverageFile)
}
```

## Coverall.io
 
 [Coverall.io](https://coveralls.io/) is a web service to help you track your code coverage over time, and ensure that 
all your new code is fully covered. For example, see [coverall maven plugin project](https://coveralls.io/github/trautonen/coveralls-maven-plugin).
It's free for open source projects so get started today! To enable it for our project, we need to add only few lines:
```gradle
apply plugin: 'com.github.kt3k.coveralls'

coveralls {
    sourceDirs = subprojects.sourceSets.main.allSource.srcDirs.flatten()
}

tasks.coveralls {
    dependsOn(jacocoRootReport)
}
```

 Note: By default Coverall.io gradle plugin expected Jacoco test xml report located at
**projectDir/reports/jacoco/test/jacocoTestReport.xml** path. When, we defined **jacocoRootReport** task, we specified 
**xml.destination** report property. Also, to be able to view coverage line be line on raw source files, we should 
provide all subprojects sources.
 
## Final root build.gradle file:
```gradle

repositories {
    jcenter()
}

buildscript {

    repositories {
        jcenter()
    }
    
    dependencies {
        classpath("org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlinVersion")
        classpath("org.kt3k.gradle.plugin:coveralls-gradle-plugin:2.8.2")
    }
}

allprojects {

    apply plugin: "jacoco"

    jacoco {
        toolVersion = "0.8.2"
    }
}

subprojects {

    apply plugin: "kotlin"
    
    test {
        useJUnitPlatform()
        jacoco {
            append = false
            destinationFile = file("$buildDir/jacoco/moduleTestsCoverage.exec")
            includeNoLocationClasses = true
            excludes = ['jdk.internal.*']
        }
    }
}


apply plugin: 'com.github.kt3k.coveralls'

def allTestsCoverageFile = "$buildDir/jacoco/rootTestsCoverage.exec"

task jacocoMergeSubprojectResultsIntoRootOne(type: JacocoMerge) {
    destinationFile = file(allTestsCoverageFile)
    executionData = project.fileTree(dir: '.', include: '**/build/jacoco/moduleTestsCoverage.exec')
}

task jacocoMerge(dependsOn: ['jacocoMergeSubprojectResultsIntoRootOne'])

task jacocoRootReport(type: JacocoReport, dependsOn: "jacocoMerge") {
    reports {
        xml.enabled = true
        html.enabled = true
        xml.destination file("${buildDir}/reports/jacoco/test/jacocoTestReport.xml")
    }
    additionalSourceDirs = files(subprojects.sourceSets.main.allSource.srcDirs)
    sourceDirectories = files(subprojects.sourceSets.main.allSource.srcDirs)
    classDirectories = files(subprojects.sourceSets.main.output)
    executionData = files(allTestCoverageFile)
}

coveralls {
    sourceDirs = subprojects.sourceSets.main.allSource.srcDirs.flatten()
}

tasks.coveralls {
    dependsOn(jacocoRootReport)
}

```


## CI
 The Coveralls service is CI-agnostic. Just add **COVERALLS_REPO_TOKEN** env variable and run
gradle coverall task:
 
```bash
./gradlew clean build coveralls
``` 



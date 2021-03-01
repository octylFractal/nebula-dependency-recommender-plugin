## Maintenance Mode Support

We don't plan add new functionality to this plugin. We focus mainly on compatibility with new version of Gradle.

Much of the functionality of this project can be replicated by a new feature coming in `Gradle 4.6` and the default in `Gradle 5.0`. We would recommend new users seeking this functionality to adopt the feature directly from the Gradle. You can find details [Sharing dependency versions between projects](https://docs.gradle.org/current/userguide/platforms.html)

We are actively working on switching from this plugin to Gradle Platform support too with eventual long term goal to deprecate this plugin.

# Nebula Dependency Recommender

![Support Status](https://img.shields.io/badge/nebula-maintence-orange.svg)
[![Gradle Plugin Portal](https://img.shields.io/maven-metadata/v/https/plugins.gradle.org/m2/com.netflix.nebula/nebula-dependency-recommender-plugin/maven-metadata.xml.svg?label=gradlePluginPortal)](https://plugins.gradle.org/plugin/nebula.dependency-recommender)
[![Maven Central](https://maven-badges.herokuapp.com/maven-central/com.netflix.nebula/nebula-dependency-recommender-plugin/badge.svg?style=plastic)](https://maven-badges.herokuapp.com/maven-central/com.netflix.nebula/nebula-dependency-recommender-plugin)
![CI](https://github.com/nebula-plugins/nebula-dependency-recommender-plugin/actions/workflows/ci.yml/badge.svg)
![Publish](https://github.com/nebula-plugins/nebula-dependency-recommender-plugin/actions/workflows/publish.yml/badge.svg)
[![Apache 2.0](https://img.shields.io/github/license/nebula-plugins/nebula-dependency-recommender-plugin.svg)](http://www.apache.org/licenses/LICENSE-2.0)

A Gradle plugin that allows you to leave off version numbers in your dependencies section and have versions recommended by several possible sources.  The most familiar recommendation provider that is supported is the Maven BOM (i.e. Maven dependency management metadata).  The plugin will control the versions of any dependencies that do not have a version specified.

Table of Contents
=================

  * [Nebula Dependency Recommender](#nebula-dependency-recommender)
    * [Usage](#usage)
    * [Dependency recommender configuration](#dependency-recommender-configuration)
    * [Built-in recommendation providers](#built-in-recommendation-providers)
    * [Producing a Maven BOM for use as a dependency recommendation source](#producing-a-maven-bom-for-use-as-a-dependency-recommendation-source)
    * [Version selection rules](#version-selection-rules)
      * [1. Forced dependencies](#1-forced-dependencies)
      * [2. Direct dependencies with a version qualifier](#2-direct-dependencies-with-a-version-qualifier)
      * [3.  Dependency recommendations](#3--dependency-recommendations)
      * [4.  Transitive dependencies](#4--transitive-dependencies)
    * [Conflict resolution and transitive dependencies](#transitive-dependencies)
    * [Accessing recommended versions directly](#accessing-recommended-versions-directly)
    * [Notes on POMs Generated by Gradle maven-publish](#9-notes-on-poms-generated-by-gradle-maven-publish)


## 1. Dependency recommender configuration

Dependency recommenders are the source of versions.  If more than one recommender defines a recommended version for a module, the last recommender specified will win.

```groovy
dependencyRecommendations {
  propertiesFile uri: 'http://somewhere/extlib.properties', name: 'myprops'
}

dependencies {
  nebulaRecommenderBom 'netflix:platform:latest.release@pom'
  implementation 'com.google.guava:guava' // no version, version is recommended
  implementation 'commons-lang:commons-lang:2.6' // I know what I want, don't recommend
  implementation project.recommend('commmons-logging:commons-logging', 'myprops') // source the recommendation from the provider named myprops'
}
```

You can also specify bom lookup via a configuration
 ```groovy
 dependencies {
   nebulaRecommenderBom 'test.nebula:bom:1.0.0@pom'
 }
 ```

## 2. Built-in recommendation providers

Several recommendation providers pack with the plugin.  The file-based providers all a shared basic configuration that is described separately.

* [File-based providers](https://github.com/nebula-plugins/nebula-dependency-recommender/wiki/File-Based-Providers)
	* [Maven BOM](https://github.com/nebula-plugins/nebula-dependency-recommender/wiki/Maven-BOM-Provider)
	* [Properties file](https://github.com/nebula-plugins/nebula-dependency-recommender/wiki/Properties-File-Provider)
	* [Nebula dependency lock](https://github.com/nebula-plugins/nebula-dependency-recommender/wiki/Dependency-Lock-Provider)
* [Map](https://github.com/nebula-plugins/nebula-dependency-recommender/wiki/Map-Provider)
* [Custom](https://github.com/nebula-plugins/nebula-dependency-recommender/wiki/Custom-Provider)

## 3. Producing a Maven BOM for use as a dependency recommendation source

Suppose you want to produce a BOM that contains a recommended version for commons-configuration.

```groovy
buildscript {
    repositories { mavenCentral() }
    dependencies { classpath 'com.netflix.nebula:nebula-dependency-recommender:4.+' }
}

apply plugin: 'maven-publish'
apply plugin: 'nebula.dependency-recommender'

group = 'netflix'

configurations { implementation }
repositories { mavenCentral() }

dependencies {
   implementation 'commons-configuration:commons-configuration:1.6'
}

publishing {
    publications {
        parent(MavenPublication) {
            // the transitive closure of this configuration will be flattened and added to the dependency management section
            nebulaDependencyManagement.fromConfigurations { configurations.implementation }

            // alternative syntax when you want to explicitly add a dependency with no transitives
            nebulaDependencyManagement.withDependencies { 'manual:dep:1' }

            // the bom will be generated with dependency coordinates of netflix:module-parent:1
            artifactId = 'module-parent'
            version = 1

            // further customization of the POM is allowed if desired
            pom.withXml { asNode().appendNode('description', 'A demonstration of maven POM customization') }
        }
    }
    repositories {
        maven {
           url "$buildDir/repo" // point this to your destination repository
        }
    }
}
```

The resultant BOM would look like this:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <modelVersion>4.0.0</modelVersion>
  <groupId>netflix</groupId>
  <artifactId>module-parent</artifactId>
  <version>1</version>
  <packaging>pom</packaging>
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>commons-digester</groupId>
        <artifactId>commons-digester</artifactId>
        <version>1.8</version>
      </dependency>
      <dependency>
        <groupId>commons-logging</groupId>
        <artifactId>commons-logging</artifactId>
        <version>1.1.1</version>
      </dependency>
      <dependency>
        <groupId>commons-lang</groupId>
        <artifactId>commons-lang</artifactId>
        <version>2.4</version>
      </dependency>
      <dependency>
        <groupId>commons-configuration</groupId>
        <artifactId>commons-configuration</artifactId>
        <version>1.6</version>
      </dependency>
      <dependency>
        <groupId>commons-beanutils</groupId>
        <artifactId>commons-beanutils</artifactId>
        <version>1.7.0</version>
      </dependency>
      <dependency>
        <groupId>commons-collections</groupId>
        <artifactId>commons-collections</artifactId>
        <version>3.2.1</version>
      </dependency>
      <dependency>
        <groupId>commons-beanutils</groupId>
        <artifactId>commons-beanutils-core</artifactId>
        <version>1.8.0</version>
      </dependency>
      <dependency>
        <groupId>manual</groupId>
        <artifactId>dep</artifactId>
        <version>1</version>
      </dependency>
    </dependencies>
  </dependencyManagement>
  <description>A demonstration of maven POM customization</description>
</project>
```

## 4. Version selection rules

The hierarchy of preference for versions is:

### 4.1. Forced dependencies

```groovy
configurations.all {
    resolutionStrategy {
        force 'commons-logging:commons-logging:1.2'
    }
}

dependencyRecommendations {
   map recommendations: ['commons-logging:commons-logging': '1.1']
}

dependencies {
   implementation 'commons-logging:commons-logging' // version 1.2 is selected
}
```

### 4.2. Direct dependencies with a version qualifier

Direct dependencies with a version qualifier trump recommendations, even if the version qualifier refers to an older version.

```groovy
dependencyRecommendations {
   map recommendations: ['commons-logging:commons-logging': '1.2']
}

dependencies {
   implementation 'commons-logging:commons-logging:1.0' // version 1.0 is selected
}
```

### 4.3.  Dependency recommendations

This is the basic case described elsewhere in the documentation;

```groovy
dependencyRecommendations {
   map recommendations: ['commons-logging:commons-logging': '1.0']
}

dependencies {
   implementation 'commons-logging:commons-logging' // version 1.0 is selected
}
```

### 4.4.  Transitive dependencies

Transitive dependencies interact with the plugin in different ways depending on which of two available strategies is selected.

#### 4.4.1.  `ConflictResolved` Strategy (default)

Consider the following example with dependencies on `commons-configuration` and `commons-logging`.  `commons-configuration:1.6` depends on `commons-logging:1.1.1`.  In this case, the transitive dependency on `commons-logging` via `commons-configuration` is conflict resolved against the recommended version of 1.0 if we have a direct on `commons-logging`.  Normal Gradle conflict resolution selects 1.1.1.

```groovy
dependencyRecommendations {
   strategy ConflictResolved // this is the default, so this line is NOT necessary
   map recommendations: ['commons-logging:commons-logging': '1.0']
}

dependencies {
   implementation 'commons-logging:commons-logging'
   implementation 'commons-configuration:commons-configuration:1.6'
}
```

#### 4.4.2.  `OverrideTransitives` Strategy

In the following example version `commons-logging:commons-logging:1.0` is selected even though `commons-logging` is not explicitly mentioned in dependencies. This would not work with the ConflictResolved strategy:

```groovy
dependencyRecommendations {
   strategy OverrideTransitives
   map recommendations: ['commons-logging:commons-logging': '1.0']
}

dependencies {
   implementation 'commons-configuration:commons-configuration:1.6'
}
```

#### 4.4.3.  Bubbling up recommendations from transitives

If no recommendation can be found in the recommendation sources for a dependency that has no version, but a version is provided by a transitive, the version provided by the transitive is applied.  In this scenario, if several transitives provide versions for the module, normal Gradle conflict resolution applies.

```groovy
dependencyRecommendations {
   map recommendations: ['some:other-module': '1.1']
}

dependencies {
   implementation 'commons-configuration:commons-configuration:1.6'
   implementation 'commons-logging:commons-logging' // version 1.1.1 is selected
}
```

## 5. Conflict resolution and transitive dependencies

* [Resolving differences between recommendation providers](https://github.com/nebula-plugins/nebula-dependency-recommender/wiki/Resolving-Differences-Between-Recommendation-Providers)

## 6. Accessing recommended versions directly

The `dependencyRecommendations` container can be queried directly for a recommended version:

```groovy
dependencyRecommendations.getRecommendedVersion('commons-logging', 'commons-logging')
```

The `getRecommendedVersion` method returns `null` if no recommendation is found.

## 7. Strict Mode

```groovy
dependencyRecommendations {
    strictMode = true
}
```

Strict mode will cause the plugin to fail if a dependency version is omitted and not found in a recommendation source.

## 8. Notes on POMs Generated by Gradle maven-publish

Gradle requires that version numbers are present in the dependencies block to create a valid POM file that includes version numbers. To fix the issue this causes when using the dependency-recommender plug-in, apply the `nebula.maven-resolved-dependencies` plug-in from the [nebula-publishing-plugin](https://github.com/nebula-plugins/nebula-publishing-plugin) set.

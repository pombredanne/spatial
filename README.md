# Neo4j Spatial

[![Build Status](https://travis-ci.org/neo4j/spatial.png)](https://travis-ci.org/neo4j/spatial)

Neo4j Spatial is a library facilitating the import, storage and querying of spatial data in the [Neo4j open source graph database](http://neo4j.org/).

This projects manual is deployed as part of the local build as the [Neo4j Spatial Manual](http://neo4j.github.io/spatial)

![Open Street Map](https://raw.github.com/neo4j/spatial/master/src/docs/images/one-street.png "Open Street Map")


Some key features include:

* Utilities for importing from ESRI Shapefile as well as Open Street Map files
* Support for all the common geometry types
* An RTree index for fast searches on geometries
* Support for topology operations during the search (contains, within, intersects, covers, disjoint, etc.) 
* The possibility to enable spatial operations on any graph of data, regardless of the way the spatial data is stored, as long as an adapter is provided to map from the graph to the geometries.
* Ability to split a single layer or dataset into multiple sub-layers or views with pre-configured filters

## Index and Querying ##


The current index is an RTree index, but it has been developed in an extensible way allowing for other indices to be added if necessary.
The spatial queries implemented are:

* Contain
* Cover
* Covered By
* Cross
* Disjoint
* Intersect
* Intersect Window
* Overlap
* Touch
* Within
* Within Distance
 
## Building ##

The simplest way to build Neo4j Spatial is by using maven. Just clone the git repository and run 

~~~bash
    mvn install
~~~

This will download all dependencies, compiled the library, run the tests and install the artifact in your local repository.

## Layers and GeometryEncoders ##

The primary type that defines a collection of geometries is the Layer. A layer contains an index for querying. In addition a Layer can be an EditableLayer if it is possible to add and modify geometries in the layer. The next most important interface is the GeometryEncoder.

The DefaultLayer is the standard layer, making use of the WKBGeometryEncoder for storing all geometry types as byte[] properties of one node per geometry instance.

The OSMLayer is a special layer supporting Open Street Map and storing the OSM model as a single fully connected graph. The set of Geometries provided by this layer includes Points, LineStrings and Polygons, and as such cannot be exported to Shapefile format, since that format only allows a single Geometry per layer. However, OMSLayer extends DynamicLayer, which allow it to provide any number of sub-layers, each with a specific geometry type and in addition based on a OSM tag filter. For example you can have a layer providing all cycle paths as LineStrings, or a layer providing all lakes as Polygons. Underneath these are all still backed by the same fully connected graph, but exposed dynamically as apparently separate geometry layers.

## Examples ##

### Importing a shapefile ###

Spatial data is divided in Layers and indexed by a RTree.

~~~java
    GraphDatabaseService database = new EmbeddedGraphDatabase(storeDir);
    try {
        ShapefileImporter importer = new ShapefileImporter(database);
        importer.importFile("roads.shp", "layer_roads");
    } finally {
        database.shutdown();
    }
~~~

### Importing an Open Street Map file ###

This is more complex because the current OSMImporter class runs in two phases, the first requiring a batch-inserter on the database. The is ongoing work to allow for a non-batch-inserter on the entire process, and possibly when you have read this that will already be available. Refer to the unit tests in classes TestDynamicLayers and TestOSMImport for the latest code for importing OSM data. At the time of writing the following worked:

~~~java
    OSMImporter importer = new OSMImporter("sweden");
    Map<String, String> config = new HashMap<String, String>();
    config.put("neostore.nodestore.db.mapped_memory", "90M" );
    config.put("dump_configuration", "true");
    config.put("use_memory_mapped_buffers", "true");
    BatchInserter batchInserter = new BatchInserterImpl(dir, config);
    importer.importFile(batchInserter, "sweden.osm", false);
    batchInserter.shutdown();

    GraphDatabaseService db = new EmbeddedGraphDatabase(dir);
    importer.reIndex(db, 10000);
    db.shutdown();
~~~

### Executing a spatial query ###

~~~java
    GraphDatabaseService database = new EmbeddedGraphDatabase(storeDir);
    try {
    	SpatialDatabaseService spatialService = new SpatialDatabaseService(database);
        Layer layer = spatialService.getLayer("layer_roads");
        SpatialIndexReader spatialIndex = layer.getIndex();
        	
        Search searchQuery = new SearchIntersectWindow(new Envelope(xmin, xmax, ymin, ymax));
        spatialIndex.executeSearch(searchQuery);
   	List<SpatialDatabaseRecord> results = searchQuery.getResults();
    } finally {
	database.shutdown();
    }
~~~

Refer to the test code in the LayerTest and the SpatialTest classes for more examples of query code. Also review the classes in the org.neo4j.gis.spatial.query package for the full range or search queries currently implemented.

## Neo4j Spatial Geoserver Plugin ##

*IMPORTANT*
Tested with: GeoServer 2.1.1

### Building ###

~~~bash
    mvn clean install
~~~


### Building and deploying the docs###

Add your Github credentials in your `~/.m2/settings.xml`

~~~xml
<settings>
    <servers>
      <server>
        <id>github</id>
        <username>xxx@xxx.xx</username>
        <password>secret</password>
      </server>
    </servers>
</settings>
~~~

now do

~~~bash
    mvn clean install site -Pneo-docs-build -
~~~

### Deployment into Geoserver ###

* unzip the `target/xxxx-server-plugin.zip` and the Neo4j libraries from your Neo4j download under `$NEO4J_HOME/lib` into `$GEOSERVER_HOME/webapps/geoserver/WEB-INF/lib`

* restart geoserver

* configure a new workspace

* configure a new datasource neo4j in your workspace. Point the "The directory path of the Neo4j database:" parameter to the relative (form the GeoServer working dir) or aboslute path to a Neo4j Spatial database with layers (see [Neo4j Spatial](https://github.com/neo4j/spatial)

* in Layers, do "Add new resource" and choose your Neo4j datastore to see the exisitng Neo4j Spatial layers and add them.

### Testing in GeoServer trunk ###

* check out the geoserver source

~~~bash
    svn co https://svn.codehaus.org/geoserver/trunk geoserver-trunk
~~~

* build the source

~~~bash
    cd geoserver-trunk
    mvn clean install
~~~

* check that you can run the web app as of [The GeoServer Maven build guide](http://docs.geoserver.org/latest/en/developer/maven-guide/index.html#running-the-web-module-with-jetty)

~~~bash
    cd src/web/app
    mvn jetty:run
~~~

* in `$GEOSERVER_SOURCE/src/web/app/pom.xml` https://svn.codehaus.org/geoserver/trunk/src/web/app/pom.xml, add the following lines under the profiles section:

~~~xml
    <profile>
      <id>neo4j</id>
      <dependencies>
        <dependency>
          <groupId>org.neo4j</groupId>
          <artifactId>neo4j-spatial</artifactId>
          <version>0.9-SNAPSHOT</version>
        </dependency>
      </dependencies>
    </profile>
~~~

* start the GeoServer webapp again with the added neo4j profile

~~~bash
    cd $GEOSERVER_SRC/src/web/app
    mvn jetty:run -Pneo4j
~~~

* find Neo4j installed as a datasource under http://localhost:8080


## Using Neo4j Spatial with uDig ##

For more info head over to [Neo4j Wiki on uDig](http://wiki.neo4j.org/content/Neo4j_Spatial_in_uDig)

## Using the Neo4j Spatial Server plugin ##

Neo4j Spatial is also packaged as a ZIP file that can be unzipped into the Neo4j Server /plugin directory. After restarting the server, you should be able to do things liek the following REST calls (here illustrated using `curl`)

Precompiled versions of that ZIP file ready for download and use:

* [for Neo4j 1.9](http://dist.neo4j.org.s3.amazonaws.com/spatial/neo4j-spatial-0.11-neo4j-1.9-server-plugin.zip)
* [for Neo4j 1.8.2](http://dist.neo4j.org.s3.amazonaws.com/spatial/neo4j-spatial-0.9.1-neo4j-1.8.2-server-plugin.zip)
* [for Neo4j 2.0-M06](http://dist.neo4j.org/spatial/neo4j-spatial-0.12-neo4j-2.0.0-M06-server-plugin.zip)

~~~bash
    #install the plugin
    cp $NEO4J_SPATIAL_HOME/target/neo4j-spatial-XXXX-server-plugin.zip $NEO4J_HOME/plugins
    cd unzip neo4j-spatial-XXXX-server-plugin.zip -d $NEO4J_HOME/plugins
    
    #start the server
    $NEO4J_HOME/bin/neo4j start

    curl http://localhost:7474/db/data/
~~~

~~~json
    {
      "relationship_index" : "http://localhost:7474/db/data/index/relationship",
      "node" : "http://localhost:7474/db/data/node",
      "relationship_types" : "http://localhost:7474/db/data/relationship/types",
      "extensions_info" : "http://localhost:7474/db/data/ext",
      "node_index" : "http://localhost:7474/db/data/index/node",
      "reference_node" : "http://localhost:7474/db/data/node/0",
      "extensions" : {
      "SpatialPlugin" : {
          "addSimplePointLayer" : "http://localhost:7474/db/data/ext/SpatialPlugin/graphdb/addSimplePointLayer",
          "addNodeToLayer" : "http://localhost:7474/db/data/ext/SpatialPlugin/graphdb/addNodeToLayer"
        }
      }
    }
~~~

~~~bash  
    curl -d "layer=test" http://localhost:7474/db/data/ext/SpatialPlugin/graphdb/addSimplePointLayer

    Creating new layer 'test' unless it already exists
~~~

~~~json
    [ {
      "outgoing_relationships" : "http://localhost:7474/db/data/node/2/relationships/out",
      "data" : {
        "layer_class" : "org.neo4j.gis.spatial.EditableLayerImpl",
        "layer" : "test",
        "geomencoder" : "org.neo4j.gis.spatial.encoders.SimplePointEncoder",
        "ctime" : 1304444390349
      },
      "traverse" : "http://localhost:7474/db/data/node/2/traverse/{returnType}",
      "all_typed_relationships" : "http://localhost:7474/db/data/node/2/relationships/all/{-list|&|types}",
      "property" : "http://localhost:7474/db/data/node/2/properties/{key}",
      "self" : "http://localhost:7474/db/data/node/2",
      "properties" : "http://localhost:7474/db/data/node/2/properties",
      "outgoing_typed_relationships" : "http://localhost:7474/db/data/node/2/relationships/out/{-list|&|types}",
      "incoming_relationships" : "http://localhost:7474/db/data/node/2/relationships/in",
      "extensions" : {
      },
      "create_relationship" : "http://localhost:7474/db/data/node/2/relationships",
      "all_relationships" : "http://localhost:7474/db/data/node/2/relationships/all",
      "incoming_typed_relationships" : "http://localhost:7474/db/data/node/2/relationships/in/{-list|&|types}"
    } ]
~~~

## Building Neo4j spatial ##

~~~bash  
    git clone https://github.com/neo4j/spatial.git
    cd spatial
    mvn clean package
~~~

### Building Neo4j Spatial Documentation ###

~~~bash  
    git clone https://github.com/neo4j/spatial.git
    cd spatial
    mvn clean install -Pneo-docs-build
~~~

## Using Neo4j spatial in your Java project with Maven ##
Add the following dependency to your project's pom.xml:

~~~xml
        <dependency>
            <groupId>org.neo4j</groupId>
            <artifactId>neo4j-spatial</artifactId>
            <version>0.9-SNAPSHOT</version>
        </dependency>
~~~

Note: neo4j-spatial has a mandatory dependency on GremlinGroovyPipeline from the
com.tinkerpop.gremlin.groovy package. The dependency in neo4j is type 'provided', so when using
neo4j-spatial in your own Java project, make sure to add the following dependency to your pom.xml, too.

~~~xml
        <dependency>
            <groupId>com.tinkerpop.gremlin</groupId>
            <artifactId>gremlin-groovy</artifactId>
            <version>1.4</version>
            <type>jar</type>
            <exclusions>
                <exclusion>
                    <groupId>com.tinkerpop.blueprints</groupId>
                    <artifactId>blueprints-sail-graph</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.codehaus.groovy.maven</groupId>
                    <artifactId>gmaven-plugin</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>jline</groupId>
                    <artifactId>jline</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
~~~

If the dependency is omitted, neo4j-spatial will probably throw an exception like this:

~~~
java.lang.NoClassDefFoundError: com/tinkerpop/gremlin/groovy/GremlinGroovyPipeline
at java.lang.ClassLoader.defineClass1(Native Method)
at java.lang.ClassLoader.defineClass(ClassLoader.java:634)
at java.security.SecureClassLoader.defineClass(SecureClassLoader.java:142)
at java.net.URLClassLoader.defineClass(URLClassLoader.java:277)
at java.net.URLClassLoader.access$000(URLClassLoader.java:73)
at java.net.URLClassLoader$1.run(URLClassLoader.java:212)
at java.security.AccessController.doPrivileged(Native Method)
at java.net.URLClassLoader.findClass(URLClassLoader.java:205)
at java.lang.ClassLoader.loadClass(ClassLoader.java:321)
at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:294)
at java.lang.ClassLoader.loadClass(ClassLoader.java:266)
at org.neo4j.gis.spatial.indexprovider.LayerNodeIndex.query(LayerNodeIndex.java:237)
~~~

## Running Neo4j spatial code from the command-line ##

Some of the classes in Neoj4-Spatial include main() methods and can be run on the command-line.
For example there are command-line options for importing SHP and OSM data. See the main methods
in the OSMImporter and ShapefileImporter classes. Here we will describe how to setup the dependencies
for running the command-line, using the OSMImporter and the sample OSM file two-street.osm.
We will show two ways to run this on the command line, one with the java command itself, and the
other using the 'exec:java' target in maven. In both cases we use maven to setup the dependencies.

### Compile ###

~~~bash
    git clone git://github.com/neo4j/spatial.git
    cd spatial
    mvn clean compile
~~~

### Run using JAVA command ###

~~~bash
    mvn dependency:copy-dependencies
    java -cp target/classes:target/dependency/* org.neo4j.gis.spatial.osm.OSMImporter osm-db two-street.osm 
~~~

The first command above only needs to be run once, to get a copy of all required JAR files into the directory target/dependency.
Once this is done, all further java commands with the -cp specifying that directory will load all dependencies.
It is likely that the specific command being run does not require all dependencies copied, since it will only be using
parts of the Neo4j-Spatial library, but working out exactly which dependencies are required can take a little time, so
the above approach is most certainly the easiest way to do this.

### Run using 'mvn exec:java' ###

~~~bash
    mvn exec:java -Dexec.mainClass=org.neo4j.gis.spatial.osm.OSMImporter -Dexec.args="osm-db two-street.osm"
~~~

Note that the OSMImporter cannot re-import the same data multiple times,
so you need to delete the database between runs if you are planning to do that.

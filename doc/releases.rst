.. _releases:

GeoScript Groovy Releases
=========================

1.3
---

    The 1.3 release of GeoScript is built on Groovy 2.1.9, GeoTools 11.0, and the Java Topology Suite 1.13.

    **Layer Geoprocessing and Layer Algebra**

        * **Layer Geoprocessing**

            * Split by Field

              Split a Layer into multiple Layers using the value of a Field::

                Memory workspace = new Memory()
                layer.split(layer.schema.get("col"), workspace)

            * Split by Layer

              Split a Layer into multiple Layers based on the Features from the split Layer::

                Memory workspace = new Memory()
                layer.split(splitLayer,splitLayer.schema.get("row_col"),workspace)

            * Buffer

              Buffer all of the Features in the Layer.  The buffer distance can be a geoscript.filter.Expression or a double.
              This allows variable distance buffers that depend on the value of a Field, a Function, or an Expression::

                layer.buffer(2)

                layer.buffer(new geoscript.filter.Property("col"))

                layer.buffer(geoscript.filter.Expression.fromCQL("col * 2"))

                layer.buffer(new geoscript.filter.Function("calc_buffer(row,col)", {row, col -> row + col}))

            * Merge

              Merge a Layer with another Layer to create an output Layer.

            * Dissolve

              Dissolve the Features of a Layer by a Field or dissolve intersecting Features of a Layer.

        * **Layer Algebra**

          The layer algebra methods were inspired by similar work done by the GDAL developers. The following
          examples use the GDAL dataset.

            .. image:: images/la_layers.png

            * clip::

                layerA.clip(layerB)

              .. image:: images/la_clip_a_b.png

            * union::

                layerA.union(layerB)

              .. image:: images/la_union.png

            * intersection::

                layerA.intersection(layerB)

              .. image:: images/la_intersection.png

            * erase::

                layerA.erase(layerB)

              .. image:: images/la_erase_a_b.png

            * identify::

                layerA.identity(layerB)

              .. image:: images/la_identity_a_b.png

            * update::

                layerA.update(layerB)

              .. image:: images/la_update_a_b.png

            * symDifference::

                layerA.symDifference(layerB)

              .. image:: images/la_symdifference.png

    **Add batches of Features to a Layer**

        * The geoscript.layer.Writer class adds batches of Features to a Layer with a Transaction::

            Writer writer = new Writer(layer, batch: 1000, transaction: 'default')
            try {
                Feature f = writer.newFeature
                writer.add(f)
            } finally {
                writer.close()
            }

            Writer writer = Writer.write(layer, batch: batch) { writer ->
                pts.eachWithIndex{Point pt, int i ->
                    Feature f = writer.newFeature
                    f.geom = pt
                    f['id'] = i
                    writer.add(f)
                }
            }

        * GeoScript Layers have a getWriter() and withWriter() methods::

            Writer writer = layer.getWriter(autoCommit: false, batch: 75)
            try {
                pts.eachWithIndex{Point pt, int i ->
                    writer.add(s.feature([the_geom: pt, id: i], "point${i}"))
                }
            } finally {
                writer.close()
            }

            layer.withWriter(batch: 45) {Writer writer ->
                pts.eachWithIndex{Point pt, int i ->
                    writer.add(s.feature([the_geom: pt, id: i], "point${i}"))
                }
            }

    **Database Workspace**

        * Improve SQL view layers by introducing **createView** and deprecating **addSqlQuery**::

            Layer layer = h2.createView("state","SELECT * FROM \"states\" WHERE \"STATE_ABBR\" = '%abbr%'",
                new Field("the_geom","Polygon","EPSG:4326"),
                params: [['abbr', 'TX']])

        * Add groovy.sql.Sql access for all Database based Workspace with the **getSql()** method.
          This allows you to do arbitray SQL queries::

            H2 h2 = new H2(folder.newFile("h2.db"))
            Layer l = h2.create('widgets',[new Field("geom", "Point"), new Field("name", "String")])
            l.add([new Point(1,1), "one"])
            l.add([new Point(2,2), "two"])
            l.add([new Point(3,3), "three"])

            // Get groovy.sql.Sql
            def sql = h2.sql

            // Count rows
            assertEquals 3, sql.firstRow("SELECT COUNT(*) as count FROM \"widgets\"").get("count") as int

            // Query
            List names = []
            sql.eachRow "SELECT \"name\" FROM \"widgets\" ORDER BY \"name\" DESC", {
                names.add(it["name"])
            }
            println names

            // Insert
            sql.execute("INSERT INTO \"widgets\" (\"geom\", \"name\") VALUES (ST_GeomFromText('POINT (6 6)',4326), 'four')")

            // Query
            sql.eachRow "SELECT ST_Buffer(\"geom\", 10) as buffer, \"name\" FROM \"widgets\"", {row ->
                Geometry poly = Geometry.fromWKB(row.buffer as byte[])
                assertNotNull poly
                assertTrue poly instanceof Polygon
                assertNotNull row.name
            }

            h2.close()

        * The H2 Workspace can connect to H2 databases using server mode::

            H2 h2 = new H2("database", "localhost", "5432", "public", "sa", "supersecret")

        * JNDI support for PostGIS, MySQL, H2::

            PostGIS postgis = new PostGIS("java:comp/env/jdbc/geoscript", schema: "public")

        * PostGIS can create or drop database::

            PostGIS postgis = new PostGIS("database", createDatabase: true, createDatabaseParams: "")

        * Database Workspaces can create, delete, list indexes::

            // Add two indexes
            h2.createIndex("widgets","geom_idx","geom",false)
            h2.createIndex("widgets","name_idx","name",true)

            // Get the indexes
            List indexes = h2.getIndexes("widgets")

            // Delete the geom index
            h2.deleteIndex("widgets","geom_idx")

        * Database Workspace can remove layers::

            h2.remove("points")

    **Raster**

        * NetCDF Raster support::

            NetCDF netcdf = new NetCDF(file)
            netcdf.names.each{ String name ->
                Raster raster = netcdf.read(name)
                println raster.bounds
                raster.dispose()
            }

        * API Change to Raster/Format API

          In order to support NetCDF Rasters, the Raster Format API was changed.  Contructors with a File or other way to connect to Rasters,
          write methods that contain the destination, or read methods that contain the source have all been deprecated and will be removed in
          the next release.  Instead, use contructors that contain a source or destination File, and read and write methods that take an optional
          Raster name (in order to support Formats that can contain more than one Raster such as NetCDF).

          Instead of::

            GeoTIFF geotiff = new GeoTIFF()
            Raster raster = geotiff.read(new File("world.tiff"))
            geotiff.write(raster.crop(new Bounds(10,10,50,50)), new File("cropped_world.tiff"))

          Please use the new API::

            GeoTIFF geotiff = new GeoTIFF(new File("world.tiff"))
            Raster raster = geotiff.read()
            new GeoTIFF(new File("cropped_world.tiff")).write(raster.crop(new Bounds(10,10,50,50)))

        * Raster.crop(Geometry)::

            GeoTIFF geoTIFF = new GeoTIFF(new File("alki.tiff"))
            Raster raster = geoTIFF.read()

            Geometry geometry = new Point(1166761.4391797914, 823593.195575958).buffer(400)
            Raster cropped = raster.crop(geometry)

    **IO Readers/Writers**

        * GPX Geometry::

            GpxReader reader = new GpxReader()
            Geometry g = reader.read("<wpt lat='2.0' lon='1.0'/>")
            assert "POINT (1 2)" == g.wkt

            GpxWriter writer = new GpxWriter()
            assert "<wpt lat='2.0' lon='1.0'/>" == writer.write(new Point(1, 2))

          GPX Feature::

            String gpx = """<wpt lat="0.0" lon="0.0">
            <name>1</name>
            <desc>This is feature # 1</desc>
            <type>Trail</type>
            <ele>45.2</ele>
            <time>1/20/14 1:47 PM</time>
            </wpt>"""
            GpxReader reader = new GpxReader()
            Feature feature = reader.read(gpx)

            GpxWriter writer = new GpxWriter(
                    name: new Property("id"),
                    time: "1/20/14 1:47 PM",
                    description: { Feature f -> "This is feature #${f['id']}" },
                    type: "Trail"
            )
            String gpx = writer.write(feature)
            assert gpx == "<wpt lat='0.0' lon='0.0' xmlns='http://www.topografix.com/GPX/1/1'>" +
                "<name>1</name><desc>This is feature #1</desc>" +
                "<type>Trail</type><time>1/20/14 1:47 PM</time></wpt>"

          GPX Layer::

            String gpx = """<?xml version="1.0" encoding="UTF-8"?>
                <gpx xmlns="http://www.topografix.com/GPX/1/1" version="1.1" creator="geoscript">
                <wpt lat="0.0" lon="0.0">
                <name>1</name>
                <desc>This is feature # 1</desc>
                <type>Trail</type>
                <ele>45.2</ele>
                <time>1/20/14 1:47 PM</time>
                </wpt>
                </gpx>"""
            GpxReader reader = new GpxReader(type: GpxReader.Type.WayPoints)
            Layer layer = reader.read(gpx)

            GpxWriter writer = new GpxWriter(
                name: new Property("id"),
                time: "1/20/14 1:47 PM",
                description: {Feature f -> "This is feature # ${f['id']}"},
                type: "Trail"
            )
            String gpx = writer.write(layer)

        * Kml IO rewritten to use Groovy's markup builder

          Geometry::

            KmlWriter writer = new KmlWriter()
            Point p = new Point(111,-47)
            assert "<Point><coordinates>111.0,-47.0</coordinates></Point>" == writer.write(p)

            KmlReader reader = new KmlReader()
            Point pt = reader.read("<Point><coordinates>111.0,-47.0</coordinates></Point>")
            assert "POINT (111 -47)" == pt.wkt

          Feature::

            String kml = """<kml:Placemark xmlns:kml="http://earth.google.com/kml/2.1" id="house1">
            <kml:name>House</kml:name>
            <kml:Point>
            <kml:coordinates>111.0,-47.0</kml:coordinates>
            </kml:Point>
            </kml:Placemark>"""
            KmlReader reader = new KmlReader()
            Feature f = reader.read(kml)

            Schema schema = new Schema("houses", [new Field("geom","Point"), new Field("name","string"), new Field("price","float")])
            Feature feature = new Feature([new Point(111,-47), "House", 12.5], "house1", schema)
            KmlWriter writer = new KmlWriter()
            assert """<kml:Placemark xmlns:kml="http://earth.google.com/kml/2.1" id="house1">
            <kml:name>House</kml:name>
            <kml:Point>
            <kml:coordinates>111.0,-47.0</kml:coordinates>
            </kml:Point>
            </kml:Placemark>""" == writer.write(feature)

          Layer::

            String kml = """<kml:kml xmlns:kml="http://earth.google.com/kml/2.1">
                <kml:Document id="featureCollection">
                    <kml:Placemark id="fid--259df7e1_131b6de0b8f_-8000">
                        <kml:name>House</kml:name>
                        <kml:Point>
                            <kml:coordinates>111.0,-47.0</kml:coordinates>
                        </kml:Point>
                    </kml:Placemark>
                    <kml:Placemark id="fid--259df7e1_131b6de0b8f_-7fff">
                        <kml:name>School</kml:name>
                        <kml:Point>
                            <kml:coordinates>121.0,-45.0</kml:coordinates>
                        </kml:Point>
                    </kml:Placemark>
                </kml:Document>
            </kml:kml>"""
            KmlReader reader = new KmlReader()
            Layer layer = reader.read(kml)

            Schema schema = new Schema("houses", [new Field("geom", "Point"), new Field("name", "string"), new Field("price", "float")])
            Memory memory = new Memory()
            Layer layer = memory.create(schema)
            layer.add(new Feature([new Point(111, -47), "House", 12.5], "house1", schema))
            layer.add(new Feature([new Point(121, -45), "School", 22.7], "house2", schema))
            KmlWriter writer = new KmlWriter()

        * GeoRSS IO using Groovy's markup builder and xml parser

          Geometry::

            GeoRSSReader reader = new GeoRSSReader()
            Point p = reader.read("<georss:point>45.256 -71.92</georss:point>")
            assert "POINT (-71.92, 45.256)" == p.wkt

            GeoRSSWriter writer = new GeoRSSWriter()
            Point p = new Point(-71.92, 45.256)
            assert "<georss:point>45.256 -71.92</georss:point>" == writer.write(p)

          Feature::

            GeoRSSReader reader = new GeoRSSReader()
            String str = "<entry xmlns:georss='http://www.georss.org/georss' xmlns='http://www.w3.org/2005/Atom'>" +
                "<title>house1</title>" +
                "<summary>[geom:POINT (111 -47), name:House, price:12.5]</summary>" +
                "<updated>12/7/2013</updated>" +
                "<georss:point>-47.0 111.0</georss:point>" +
                "</entry>"
            Feature feature = reader.read(str)

            GeoRSSWriter writer = new GeoRSSWriter(feedType: "atom", geometryType: "gml", itemDate: "12/7/2013")
            assert "<entry xmlns:georss='http://www.georss.org/georss' xmlns='http://www.w3.org/2005/Atom' " +
                "xmlns:gml='http://www.opengis.net/gml'>" +
                "<title>house1</title>" +
                "<summary>[geom:POINT (111 -47), name:House, price:12.5]</summary>" +
                "<updated>12/7/2013</updated>" +
                "<georss:where><gml:Point><gml:pos>-47.0 111.0</gml:pos></gml:Point></georss:where>" +
                "</entry>" == writer.write(feature)

          Layer::

            GeoRSSReader reader = new GeoRSSReader()
            Layer layer = reader.read("""<?xml version="1.0" encoding="utf-8"?>
             <feed xmlns="http://www.w3.org/2005/Atom"
                   xmlns:georss="http://www.georss.org/georss">
               <title>Earthquakes</title>
               <subtitle>International earthquake observation labs</subtitle>
               <link href="http://example.org/"/>
               <updated>2005-12-13T18:30:02Z</updated>
               <author>
                  <name>Dr. Thaddeus Remor</name>
                  <email>tremor@quakelab.edu</email>
               </author>
               <id>urn:uuid:60a76c80-d399-11d9-b93C-0003939e0af6</id>
               <entry>
                  <title>M 3.2, Mona Passage</title>
                  <link href="http://example.org/2005/09/09/atom01"/>
                  <id>urn:uuid:1225c695-cfb8-4ebb-aaaa-80da344efa6a</id>
                  <updated>2005-08-17T07:02:32Z</updated>
                  <summary>We just had a big one.</summary>
                  <georss:box>42.943 -71.032 43.039 -69.856</georss:box>
               </entry>
             </feed>""")

             GeoRSSWriter writer = new GeoRSSWriter(
                feedType: "atom",
                geometryType: "simple",
                itemDate: "1/22/1975",
                itemTitle: new Property("name"),
                itemDescription: { Feature f ->
                    f['description']
                }
            )
            Schema schema = new Schema("points", [
                ["geom", "Point"],
                ["name", "string"],
                ["description", "string"],
                ["id", "int"]
            ])
            Workspace workspace = new Memory()
            Layer layer = workspace.create(schema)
            layer.withWriter { writer ->
                writer.add(schema.feature([geom: "POINT (1 1)", name: "Washington", description: "The state of Washington", id: 1], "state.1"))
                writer.add(schema.feature([geom: "POINT (2 2)", name: "Oregon", description: "The state of Oregon", id: 2], "state.2"))
                writer.add(schema.feature([geom: "POINT (3 3)", name: "California", description: "The state of California", id: 3], "state.3"))
            }
            println writer.write(createLayer())

        * geoscript.layer.io.GeoJSONReader supports reading features that have different schemas

        * geoscript.feature.Feature now has getGeoJSON(), getGeoRSS(), getKml(), getGml() methods

        * Removed JDOM dependency with Groovy's native XML support

        * Removed org.json dependency with GeoTools GeoJSON support

    **Rendering**

        * Randomized Fill::

            import geoscript.layer.Shapefile
            import geoscript.render.Draw
            import geoscript.style.*

            shp = new Shapefile("states.shp")
            shp.style = (new Fill(null).hatch("circle", new Fill("#aaaaaa"), 1).random([random: "free", symbolCount: "50", tileSize: "100"]).where("PERSONS < 2000000")) +
                    (new Fill(null).hatch("circle", new Fill("#aaaaaa"), 2).random([random: "free", symbolCount: "200", tileSize: "100"]).where("PERSONS BETWEEN 2000000 AND 4000000")) +
                    (new Fill(null).hatch("circle", new Fill("#aaaaaa"), 2).random([random: "free", symbolCount: "700", tileSize: "100"]).where("PERSONS > 4000000")) +
                    (new Stroke("black", 0.1) + new Label(property: "STATE_ABBR", font: new Font(family: "Times New Roman", style: "normal", size: 14)).point([0.5, 0.5]).halo(new Fill("#FFFFFF"), 2))

            println shp.style.sld
            Draw.draw(shp)

          .. image:: images/randomized_fill.png

        * Hatch can take fill and stroke::

            Hatch hatch = new Hatch("circle", new Fill("red"), new Stroke("wheat",0.1), 10)

        * geoscript.render.Draw now accepts an optional backgroundColor parameter::

            Symbolizer sym = new Stroke('black', 2) + new Fill('gray',0.75)
            Geometry geom = new Point(0,0).buffer(0.2)
            draw(geom, style: sym, bounds: geom.bounds.scale(1.1), size: [250,250], format: "png", backgroundColor: "white")

        * geoscript.render.Map is updated and deprecated class have been removed. This was contributed by Scott Bortman.  Thanks Scott!

    **API Updates**

        * GeometryCollections now have a slice method that takes a start index and an optional end index::

            import geoscript.geom.*
            GeometryCollection g = Geometry.fromWKT("MULTIPOINT ((1 1), (2 2), (3 3), (4 4), (5 5))")
            assert "MULTIPOINT ((2 2), (3 3))" == g.slice(1,3).wkt

          When the end index is absent it defaults to the end of the collection::

            assert "MULTIPOINT ((3 3), (4 4), (5 5))" == g.slice(2).wkt

          Both the start and end index may be negative::

            assert "MULTIPOINT ((3 3), (4 4), (5 5))" == g.slice(-3).wkt
            assert "MULTIPOINT ((2 2), (3 3))" == g.slice(-4, -2).wkt

        * Get the angle between this Point and another Point::

            assert 45 == new Point(0,0).getAngle(new Point(10,10))

            assert -135, new Point(0,0).getAngle(new Point(-10,-10), "degrees")

            assert 2.3561 == new Point(0,0).getAngle(new Point(-10,10), "radians")

        * Get the azimuth between this Point and the other Point::

            assert 44.75 == new Point(0,0).getAzimuth(new Point(10,10))

            assert 135.24 == new Point(0,0).getAzimuth(new Point(10,-10))

        * Fields now have a isGeometry() method.

        * You can set the values of a Feature by passing in a Map::

            feature.set([price: 1200.5, name: "Car"])

            feature.set(price: 12.2, name: "Book")

        * Or by passing in an existing Feature::

            Feature feature = schema.feature([geom: new Point(121,-49), price: 15.6, name: "Test"])
            newFeature.set(feature)

        * Schema now has a way to create new Features with default values::

            Feature f = schema.feature()

        * Schema can also create new Features from an existing Feature::

            Feature f = schema.feature(existingFeature)

        * When a Schema creates a Feature, the default ID is now created by the GeoTools SimpleFeatureBuilder's createDefaultFeatureId() method.

    **Command line programs**

        * Add -Dorg.geotools.referencing.forceXY=true to all command line programs

1.2
---

    The 1.2 release of GeoScript was built on Groovy 2.1.6, GeoTools 10.0, and Java Topology Suite 1.13.

    The focus was on upgrading to a modern and supported version of Groovy and a few small features and bug fixes.

    **Upgrades**

        * Upgrade to GeoTools 10

        * Upgrade to Groovy 2.1.6

        * Upgrade to GeoCSS 0.8.3

    **Features**

        * The geoscript.layer.io.Readers can now take optional projection, workspace, name parameters

        * The geoscript.layer.io.CsvReader and CsvWriter by default now encode Field type in the header but this can be disabled

        * Added a MultiLineString.polygonizeFull() method that returns a Map with polygons, cut edges, dangles, and invalid ring lines.

        * Added Schema.includeFields to create a new Schema from an existing Schema with a subset of fields

    **Bug Fixes**

        * Fixed Cursor paging bug - it's start and max not start and end

        * Fixed CsvReader bug couldn't guess WKT when it was formatted without a space ("POINT(1 1)" instead of "POINT (1 1)")

1.1.1
-----

    The 1.1.1 release of GeoScript Groovy just fixes a few minor bugs.

    **Bug Fixes**

    * Fixed Cursor paging bug - it's start and max not start and end

    * Fixed CsvReader bug couldn't guess WKT when it was formatted without a space ("POINT(1 1)" instead of "POINT (1 1)")

1.1
---

    The 1.1 release of GeoScript was built on Groovy 1.8.9, GeoTools 9.x, and Java Topology Suite (JTS) 1.13

    The focus was on adding a Raster support (geoscript.layer), Charting (geoscript.plot), and numerous bug fixes and small features.

    **Raster**

        * Format (ArcGri, GeoTIFF, GTopo30, Grass, ImagePyramid, MrSID, WorldImage)

          Formats allow you to read and write Rasters::

            import geoscript.layer.*

            def format = new GeoTIFF()
            def raster = format.read(new File("raster.tif"))

            def format2 = new WorldImage()
            format2.write(raster, new File("raster.png"))

        * Raster::

            import geoscript.layer.*

            def format = new GeoTIFF()
            def raster = format.read(new File("raster.tif"))

            println raster.proj.id
            println raster.bounds
            println raster.size

        * Band::

            import geoscript.layer.*

            def format = new GeoTIFF()
            def raster = format.read(new File("raster.tif"))
            raster.bands.eachWithIndex{b,i ->
                println "Band ${i}:"
                println "   Min: ${b.min}"
                println "   Max: ${b.max}"
                println "   NoData: ${b.noData}"
                println "   Unit: ${b.unit}"
                println "   Scale: ${b.scale}"
                println "   Offset: ${b.offset}"
                println "   Type: ${b.type}"
            }

        * MapAlgebra (which is powered by Jiffle)::

            import geoscript.layer.*

            def format = new GeoTIFF()
            def raster = format.read(new File("raster.tif"))

            Raster rasterPlusTen = raster + 10

            MapAlgebra algebra = new MapAlgebra()
            Raster output = algebra.calculate("dest = src > 200;", [src: raster], size: [600, 400])

        * Process

          Raster support was added to the Process module (geoscript.process.Process) which opens up numerous geospatial algorithms like heatmap, barnes surface, and raster algebra.::


            Process process = new Process("vec:BarnesSurface")
            results = process.execute([
                data: layer.cursor,
                valueAttr: "value",
                scale: 300,
                convergence: 0.3,
                passes: 2,
                minObservations: 1,
                maxObservationDistance: 0,
                pixelsPerCell: 1,
                noDataValue: -999,
                outputWidth: 100,
                outputHeight: 100,
                outputBBOX: layer.bounds
            ])
            Raster raster = results.result

        * Style

          Raster specific Symbolizers were added to the geoscript.style module::

            def raster = new RasterSymbolizer(0.5)

            def channel = new ChannelSelection("red", "green", "blue")

            def colorMap = new ColorMap([[color: "#008000", quantity:70], [color:"#663333", quantity:256]])

            def c = new ContrastEnhancement("histogram", 0.5)

            def shadedRelief = new ShadedRelief(35, true)

        * Rendering

          The geoscript Rendering module (geoscript.render) now supports drawing Rasters::

            import geoscript.layer.*
            import geoscript.render.*

            def format = new GeoTIFF()
            def raster = format.read(new File("raster.tif"))
            Draw.draw(raster)

            Map map = new Map(layers:[new Shapefile("states.shp"), raster])
            def image = map.drawToImage()

    **Plot**

        * Chart

          A Chart can be created by one of the factory classes (Bar, Box, Curve, Pie, Regression, and Scatter).Once created, you can display it as an interactive app, save it to a File, or save it to an Image::

            Chart chart = Box.box(["A":[1,10,20],"B":[45,39,10],"C":[2,4,9],"D":[14,15,19]])
            chart.show()
            chart.save(new File("bar.jpeg"))
            def image = chart.image

        * Bar::

            Chart chart = Bar.xy([[1,10],[45,12],[23,3],[5,20]])

            Chart chart = Bar.category(["A":20,"B":45,"C":2,"D":14])

        * Box::

            Chart chart = Box.box(["A":[1,10,20],"B":[45,39,10],"C":[2,4,9],"D":[14,15,19]])

        * Curve::
            
            Chart chart = Curve.curve([[1,10],[45,12],[23,3],[5,20]])

        * Pie::
        
            Chart chart = Pie.pie(["A":20,"B":45,"C":2,"D":14])

        * Regression::

            def points = Geometry.createRandomPoints(new Bounds(0,0,100,100).geometry, 10)
            List data = points.geometries.collect{pt ->
                [pt.x,pt.y]
            }
                
        * Scatter::

            def points = Geometry.createRandomPoints(new Bounds(0,0,100,100).geometry, 10)
            List data = points.geometries.collect{pt ->
             [pt.x,pt.y]
            }

    **General**

        * Include GroovyDocs in zip distribution

    **Geometry**

        * List<Point> getNearestPoints(Geometry other)
        * List<Point> getPoints()
        * Geometry smooth(double fit)
        * static Geometry cascadedUnion(List<Polygon> polygons)
        * static Geometry fromString(String str)

    **Polygon & MultiPolygon**

        * Geometry split(LineString lineString)
        * Geometry split(MultiLineString multiLineString)

    **Schema**

        * boolean has(def field)
        * Map addSchema(Map options = [:], Schema otherSchema, String newName)
        * Schema changeField(Field oldField, Field newField, String name)
        * Schema changeFields(Map<Field, Field> fieldsToChange, String name)
        * Schema addFields(List<Field> newFields, String name)
        * Schema addField(Field field, String name)
        * Schema removeField(Field field, String name)
        * Schema removeFields(List<Field> fieldsToRemove, String name)

    **Layer**

        * Reproject features on the fly when using a Cursor::

            Cursor c = layer.getCursor(destProj: "EPSG:2927")

        * Set source projection when reprojecting Layers::

            Layer layer2 = layer1.reproject(new Projection("EPSG:2927"), "projected_facilties", 1000, new Projection("EPSG:4326"))

        * Add a List of Maps to a Layer inside of a Transaction::

            layer1.add([
                [geom: new Point(100,-45), name: "Point 1", price: 1.0],
                [geom: new Point(101,-46), name: "Point 2", price: 10.0],
                [geom: new Point(102,-47), name: "Point 3", price: 100.0],
            ])

        * Layer.transform using gt-transform module::

            Layer layer2 = layer.transform("buffered_facilities", [
                geom: "buffer(geom, 10)",
                name: "strToUpperCase(name)",
                price: "price * 10"
            ])

        * geoscript.layer.Property::

            Property prop = new Property('states.properties')

        * Feature first(Map options = [:])

        * Layer.update can take an Expression::

            layer.update(s.get("price"), Expression.fromCQL("price * 2"))

        * Layer.update(groovy script)::

            layer.update(s.get('name'), "return c + '). ' + f.get('name')", Filter.PASS, true)

        * Layer reproject(Projection p, Workspace outputWorkspace, String newName, int chunk=1000, Projection sourceProjection = new Projection("EPSG:4326"))

          Reproject a Layer to another Layer in the given Workspace

        * Layer reproject(Layer projectedLayer, int chunk = 1000, Projection sourceProjection = new Projection("EPSG:4326"))

          Reproject a Layer to another Layer that already exists.

        * Layer getCursor(fields:[])::

            layer.getCursor([fields: ["name"]])

    **Layer IO**

        * KmlReader and KmlWriter

    **Filter**

        * Filter getNot()::

            new Filter("name='foo').not == new Filter("name<>'foot')

        * Filter.simplify()

    **Expression**

        * Object evaluate(Object obj = null)::

            Expression e = new Expression(12)
            assertEquals 12, e.evaluate()

    **Bounds**

        * void setProj(def projection)
        * static Bounds fromString(String str)
        * Bounds fixAspectRatio(int w, int h)
        * boolean contains(Bounds other)
        * double getAspect()
        * Geometry getGrid(int columns, int rows, String type = "polygon")
        * Geometry getGrid(double cellWidth, double cellHeight, String type = "polygon")
        * void generateGrid(int columns, int rows, String type, Closure c)
        * void generateGrid(double cellWidth, double cellHeight, String type, Closure c)

    **WMS**

        * WMS::

            WMS wms = new WMS("http://localhost:8080/geoserver/ows?service=wms&version=1.1.1&request=GetCapabilities")
            println "Name: ${wms.name}"
            println "Title: ${wms.title}"
            def image = wms.getImage("world:borders")

        * WMSLayer::

            WMS wms = new WMS("http://localhost:8080/geoserver/ows?service=wms&version=1.1.1&request=GetCapabilities")
            def map = new geoscript.render.Map(
                layers: [new WMSLayer(wms, ["world:borders","world:cities"])]
            )
            map.render(new File("map_world.png"))

    **Workspace**

        * WFS::

            def wfs = new WFS("http://localhost:8080/geoserver/ows?service=wfs&version=1.1.0&request=GetCapabilities", timeout: 9000)

        * H2 constructor with database file instead of directory

    **Render**

        * GIF

          Image subclass that includes animated GIF support!::

            Map map = new Map(layers: [layer], backgroundColor: "white")
            GIF gif = new GIF()
            def img = gif.render(map)

            Map map = new Map(layers: [layer], backgroundColor: "white")
            GIF gif = new GIF()
            List images = ["WA","OR","CA"].collect {state ->
              map.bounds = layer.getFeatures("STATE_ABBR = '${state}'")[0].bounds
              def image = gif.render(map)
              image
            }
            File file = File.createTempFile("image_",".gif")
            gif.renderAnimated(images, file, 500, true)

        * PNG::

            Map map = new Map(layers: [layer], backgroundColor: "white")
            PNG png = new PNG()
            def img = png.render(map)

        * JPEG::

            Map map = new Map(layers: [layer], backgroundColor: "white")
            JPEG jpeg = new JPEG()
            def img = jpeg.render(map)

1.0
---

   The 1.0 release of GeoScript was built on Groovy 1.8.8, GeoTools 8.x and Java Topology Suite (JTS) 1.12.

   The focus was the following modules:

        * Geometry (geoscript.geom)
        * Projection (geoscript.proj)
        * Vector Layers (geoscript.feature, geoscript.layer, geoscript.workspace)
        * Rendering (geoscript.viewer, geoscript.style, geoscript.render)
        * Process (geoscript.process)
        * Spatial Index (geoscript.index)
        * Expressions (geoscript.filter)

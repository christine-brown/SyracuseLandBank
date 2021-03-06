# Grocery Stores



### Data Acquisition and Preparation
Syracuse University's Geography Department has a collection of data from which the grocery data used here is pulled.  Once accessed, it must be cleaned, geocoded, and aggregated in order to be included in the analysis of all variables collected for this project. 

```r
#Load packages
library( dplyr )
library( geojsonio )
library( ggmap )
library( maps )
library( maptools )
library( raster )
library( rgdal )
library( rgeos )
library( sp )

#Get grocery data from Community Geography
dir.create( "grocery_shape" )
download.file( "http://communitygeography.org/wp-content/uploads/2015/08/Supermarkets.shp_.zip" , "grocery_shape/supermarkets.zip" )
unzip( "grocery_shape/supermarkets.zip" , exdir = "grocery_shape" )

#Change original data into shapefile and convert to CSV
grocery <- readShapePoints( fn = "grocery_shape/Supermarkets" , proj4string = CRS( "+proj=longlat +datum=WGS84" ) )
grocery <- as.data.frame( grocery , stringsAsFactors = FALSE )

#Delete Community Geography file because CSV has all information
unlink( "grocery_shape" , recursive = TRUE )

#Write data as a CSV for future access
write.csv( grocery, file = "../../DATA/RAW_DATAgrocery_raw.csv" , row.names = FALSE )

#Clean data
grocery$City <- ifelse( is.na( grocery$City ), as.character( grocery$ARC_City_1 ) , as.character( grocery$City ) )
grocery <- mutate( grocery, Address = paste( Location , City , "NY" , ZipCode , sep = ", " ) )
grocery <- grocery[ , c( "Supermarke" , "Address" ) ]

#Geocode
grocery_coordinates <- suppressMessages( geocode( grocery$Address , messaging = F ) )
grocery <- cbind( grocery, grocery_coordinates )

#Get census tract information
syr_tracts <- geojson_read( "../../SHAPEFILES/SYRCensusTracts.geojson" , method="local" , what="sp" )
syr_tracts <- spTransform( syr_tracts , CRS( "+proj=longlat +datum=WGS84" ) )

#Match to census tract
grocery_coordinates_SP <- SpatialPoints( grocery_coordinates , proj4string = CRS("+proj=longlat +datum=WGS84" ) )
grocery_tract <- over( grocery_coordinates_SP , syr_tracts )
grocery <- cbind( grocery , grocery_tract )
grocery <- filter( grocery , !is.na( INTPTLON10 ) )

#Export to CSV
write.csv( grocery, file = "../../DATA/PROCESSED_DATA/grocery_processed.csv" , row.names = FALSE )

#Aggregate data
grocery_agg <- as.data.frame( table( grocery$GEOID10 ) )
grocery_agg$YEAR <- 2015
names( grocery_agg ) <- c( "TRACT" , "GROCERY" , "YEAR" )

#Export to CSV
write.csv( grocery_agg , file = "../../DATA/AGGREGATED_DATA/grocery_aggregated.csv" , row.names = FALSE )
```

### Data Visualization
One of the more interesting metrics for food is food access. Low-food access is measured at 1/2 mile and 1 mile from a supermarket or large grocery store.  The interstates are included in orange for spatial context. 

```r
#Load road data for context
roads <- geojson_read( "../../SHAPEFILES/roads.geojson" , method = "local" , what = "sp" )
roads <- spTransform( roads, CRS( "+proj=longlat +datum=WGS84" ) )

#Subset interstates
interstate <- roads[ roads$RTTYP == "I" , ]

#Clip roads
syr_outline <- gBuffer( syr_tracts , width = .000 , byid = F )
interstate_clipped <- gIntersection( syr_outline , interstate , byid = TRUE , drop_lower_td = TRUE )

#Create buffers
syr_outline <- gBuffer( syr_tracts , width = .000 , byid = F )
buff_half <- gBuffer( grocery_coordinates_SP , width = .0095 , byid = F )
buff_half_clipped <- gIntersection( syr_outline , buff_half , byid = TRUE , drop_lower_td = T )
buff_one <- gBuffer( grocery_coordinates_SP , width = .019 , byid = F )
buff_one_clipped <- gIntersection( syr_outline , buff_one , byid = TRUE , drop_lower_td = T )

#Plot buffers
par( mar = c( 0 , 0 , 1 , 0 ) )
plot( syr_tracts , col = "gray80" , main = "Grocery Stores" )
plot( interstate_clipped , col = "#dd7804" , lwd = 1.75 , add = T )
plot( buff_one_clipped , col = rgb( 10 , 95 , 193 , 40 , maxColorValue = 255 ) , border = F , add = T )
plot( buff_half_clipped , col = rgb( 10 , 95 , 193 , 70 , maxColorValue = 255 ) , border = F , add = T )
points( grocery$lon, grocery$lat , pch = 19 , col = "#0A5F99", cex = 1.5 )
map.scale( x = -76.22 , y = 42.994 , metric = F , ratio = F , relwidth = 0.1 , cex = 1 )

#Add legend
legend( x = -76.224 , y = 43.0135 , pch = 20 , pt.cex = 1.4 , cex = .8 , xpd = NA , bty = "n" , 
        inset = -.01 , legend = c( "Grocery Store" , "1/2 Mile Radius" , "1 Mile Radius" ) , 
        col = c( "#0A5F99" , rgb( 10 , 95 , 193 , 94 , maxColorValue = 255 ) , 
                 rgb( 10 , 95 , 193 , 74 , maxColorValue = 255 ) ) )
```

![](grocery_files/figure-html/unnamed-chunk-2-1.png)<!-- -->

Systematic landcover bias in Collection 5 MODIS cloud mask and derived products – a global overview
__________


```r
opts_chunk$set(eval = F)
```


This document describes the analysis of the Collection 5 MOD35 data.

# Google Earth Engine Processing
The following code produces the annual (2009) summaries of cloud frequency from MOD09, MOD35, and MOD11 using the Google Earth Engine 'playground' API [http://ee-api.appspot.com/](http://ee-api.appspot.com/). 

```coffee
var startdate="2009-01-01"
var stopdate="2009-12-31"

// MOD11 MODIS LST
var mod11 = ee.ImageCollection("MOD11A2").map(function(img){ 
  return img.select(['LST_Day_1km'])});
// MOD09 internal cloud flag
var mod09 = ee.ImageCollection("MOD09GA").filterDate(new Date(startdate),new Date(stopdate)).map(function(img) {
  return img.select(['state_1km']).expression("((b(0)/1024)%2)");
});
// MOD35 cloud flag
var mod35 = ee.ImageCollection("MOD09GA").filterDate(new Date(startdate),new Date(stopdate)).map(function(img) {
  return img.select(['state_1km']).expression("((b(0))%4)==1|((b(0))%4)==2");
});

//define reducers
var COUNT = ee.call("Reducer.count");
var MEAN = ee.call("Reducer.mean");

//a few maps of constants
c100=ee.Image(100);  //to multiply by 100
c02=ee.Image(0.02);  //to scale LST data
c272=ee.Image(272.15); // to convert K->C

//calculate mean cloudiness (%), rename, and convert to integer
mod09a=mod09.reduce(MEAN).select([0], ['MOD09']).multiply(c100).int8();
mod35a=mod35.reduce(MEAN).select([0], ['MOD35']).multiply(c100).int8();

/////////////////////////////////////////////////
// Generate the cloud frequency surface:
getMiss = function(collection) {
  //filter by date
i2=collection.filterDate(new Date(startdate),new Date(stopdate));
// number of layers in collection
i2_n=i2.getInfo().features.length;
//get means
// image of -1s to convert to % missing
c1=ee.Image(-1);
// 1 Calculate the number of days with measurements
// 2 divide by the total number of layers
i2_c=ee.Image(i2_n).float()
// 3 Add -1 and multiply by -1 to invert to % cloudy
// 4 Rename to "Percent_Cloudy"
// 5 multiply by 100 and convert to 8-bit integer to decrease file size
i2_miss=i2.reduce(COUNT).divide(i2_c).add(c1).multiply(c1).multiply(c100).int8();
return (i2_miss);
};

// run the function
mod11a=getMiss(mod11).select([0], ['MOD11_LST_PMiss']);

// get long-term mean
mod11b=mod11.reduce(MEAN).multiply(c02).subtract(c272).int8().select([0], ['MOD11_LST_MEAN']);

// summary object with all layers
summary=mod11a.addBands(mod11b).addBands(mod35a).addBands(mod09a)

var region='[[-180, -60], [-180, 90], [180, 90], [180, -60]]'  //global

// get download link
print("All")
var path = summary.getDownloadURL({
  'scale': 1000,
  'crs': 'EPSG:4326',
  'region': region
});
print('https://earthengine.sandbox.google.com' + path);
```


# Data Processing


```r
setwd("~/acrobates/adamw/projects/MOD35C5")
library(raster)
```

```
## Loading required package: sp
```

```r
beginCluster(10)
```

```
## Loading required package: snow
```

```r
library(rasterVis)
```

```
## Loading required package: lattice Loading required package: latticeExtra
## Loading required package: RColorBrewer Loading required package: hexbin
## Loading required package: grid
```

```r
library(rgdal)
```

```
## rgdal: version: 0.8-10, (SVN revision 478) Geospatial Data Abstraction
## Library extensions to R successfully loaded Loaded GDAL runtime: GDAL
## 1.9.2, released 2012/10/08 but rgdal build and GDAL runtime not in sync:
## ... consider re-installing rgdal!! Path to GDAL shared files:
## /usr/share/gdal/1.9 Loaded PROJ.4 runtime: Rel. 4.8.0, 6 March 2012,
## [PJ_VERSION: 480] Path to PROJ.4 shared files: (autodetected)
```

```r
library(plotKML)
```

```
## plotKML version 0.3-5 (2013-05-16) URL:
## http://plotkml.r-forge.r-project.org/
## 
## Attaching package: 'plotKML'
## 
## The following object is masked from 'package:raster':
## 
## count
```

```r
library(Cairo)
library(reshape)
```

```
## Loading required package: plyr
## 
## Attaching package: 'plyr'
## 
## The following object is masked from 'package:plotKML':
## 
## count
## 
## The following object is masked from 'package:raster':
## 
## count
## 
## Attaching package: 'reshape'
## 
## The following object is masked from 'package:plyr':
## 
## rename, round_any
## 
## The following object is masked from 'package:raster':
## 
## expand
```

```r
library(rgeos)
```

```
## rgeos version: 0.2-19, (SVN revision 394) GEOS runtime version:
## 3.3.3-CAPI-1.7.4 Polygon checking: TRUE
```

```r
library(splancs)
```

```
## Spatial Point Pattern Analysis Code in S-Plus
## 
## Version 2 - Spatial and Space-Time analysis
## 
## Attaching package: 'splancs'
## 
## The following object is masked from 'package:raster':
## 
## zoom
```

```r

## get % cloudy
mod09 = raster("data/MOD09_2009.tif")
names(mod09) = "C5MOD09CF"
NAvalue(mod09) = 0

mod35c5 = raster("data/MOD35_2009.tif")
names(mod35c5) = "C5MOD35CF"
NAvalue(mod35c5) = 0

## mod35C6 annual
mod35c6 = raster("data/MOD35C6_2009.tif")
names(mod35c6) = "C6MOD35CF"
NAvalue(mod35c6) = 255

## landcover
lulc = raster("data/MCD12Q1_IGBP_2009_051_wgs84_1km.tif")

# lulc=ratify(lulc)
data(worldgrids_pal)  #load palette
IGBP = data.frame(ID = 0:16, col = worldgrids_pal$IGBP[-c(18, 19)], lulc_levels2 = c("Water", 
    "Forest", "Forest", "Forest", "Forest", "Forest", "Shrublands", "Shrublands", 
    "Savannas", "Savannas", "Grasslands", "Permanent wetlands", "Croplands", 
    "Urban and built-up", "Cropland/Natural vegetation mosaic", "Snow and ice", 
    "Barren or sparsely vegetated"), stringsAsFactors = F)
IGBP$class = rownames(IGBP)
rownames(IGBP) = 1:nrow(IGBP)
levels(lulc) = list(IGBP)
names(lulc) = "MCD12Q1"

## MOD17
mod17 = raster("data/MOD17.tif", format = "GTiff")
NAvalue(mod17) = 65535
names(mod17) = "MOD17_unscaled"

mod17qc = raster("data/MOD17qc.tif", format = "GTiff")
NAvalue(mod17qc) = 255
names(mod17qc) = "MOD17CF"

## MOD11 via earth engine
mod11 = raster("data/MOD11_2009.tif", format = "GTiff")
names(mod11) = "MOD11_unscaled"
NAvalue(mod11) = 0

mod11qc = raster("data/MOD11qc_2009.tif", format = "GTiff")
names(mod11qc) = "MOD11CF"
```


Import the Collection 5 MOD35 processing path:

```r
pp = raster("data/MOD35pp.tif")
NAvalue(pp) = 255
names(pp) = "MOD35pp"
```


Define transects to illustrate the fine-grain relationship between MOD35 cloud frequency and both landcover and processing path.

```r
r1 = Lines(list(Line(matrix(c(-61.688, 4.098, -59.251, 3.43), ncol = 2, byrow = T))), 
    "Venezuela")
r2 = Lines(list(Line(matrix(c(133.746, -31.834, 134.226, -32.143), ncol = 2, 
    byrow = T))), "Australia")
r3 = Lines(list(Line(matrix(c(73.943, 27.419, 74.369, 26.877), ncol = 2, byrow = T))), 
    "India")
r4 = Lines(list(Line(matrix(c(33.195, 12.512, 33.802, 12.894), ncol = 2, byrow = T))), 
    "Sudan")

trans = SpatialLines(list(r1, r2, r3, r4), CRS("+proj=longlat +ellps=WGS84 +datum=WGS84 +no_defs "))
### write out shapefiles of transects
writeOGR(SpatialLinesDataFrame(trans, data = data.frame(ID = names(trans)), 
    match.ID = F), "output", layer = "transects", driver = "ESRI Shapefile", 
    overwrite = T)
```


Buffer transects to identify a small region around each transect for comparison and plotting

```r
transb = gBuffer(trans, byid = T, width = 0.4)
## make polygons of bounding boxes
bb0 <- lapply(slot(transb, "polygons"), bbox)
bb1 <- lapply(bb0, bboxx)
# turn these into matrices using a helper function in splancs
bb2 <- lapply(bb1, function(x) rbind(x, x[1, ]))
# close the matrix rings by appending the first coordinate
rn <- row.names(transb)
# get the IDs
bb3 <- vector(mode = "list", length = length(bb2))
# make somewhere to keep the output
for (i in seq(along = bb3)) bb3[[i]] <- Polygons(list(Polygon(bb2[[i]])), ID = rn[i])
# loop over the closed matrix rings, adding the IDs
bbs <- SpatialPolygons(bb3, proj4string = CRS(proj4string(transb)))
```


Extract the CF and mean values from each raster of interest.

```r
trd1 = lapply(1:length(transb), function(x) {
    td = crop(mod11, transb[x])
    tdd = lapply(list(mod35c5, mod35c6, mod09, mod17, mod17qc, mod11, mod11qc, 
        lulc, pp), function(l) resample(crop(l, transb[x]), td, method = "ngb"))
    ## normalize MOD11 and MOD17
    for (j in which(do.call(c, lapply(tdd, function(i) names(i))) %in% c("MOD11_unscaled", 
        "MOD17_unscaled"))) {
        trange = cellStats(tdd[[j]], range)
        tscaled = 100 * (tdd[[j]] - trange[1])/(trange[2] - trange[1])
        tscaled@history = list(range = trange)
        names(tscaled) = sub("_unscaled", "", names(tdd[[j]]))
        tdd = c(tdd, tscaled)
    }
    return(brick(tdd))
})
## bind all subregions into single dataframe for plotting
trd = do.call(rbind.data.frame, lapply(1:length(trd1), function(i) {
    d = as.data.frame(as.matrix(trd1[[i]]))
    d[, c("x", "y")] = coordinates(trd1[[i]])
    d$trans = names(trans)[i]
    d = melt(d, id.vars = c("trans", "x", "y"))
    return(d)
}))
transd = do.call(rbind.data.frame, lapply(1:length(trans), function(l) {
    td = as.data.frame(extract(trd1[[l]], trans[l], along = T, cellnumbers = F)[[1]])
    td$loc = extract(trd1[[l]], trans[l], along = T, cellnumbers = T)[[1]][, 
        1]
    td[, c("x", "y")] = xyFromCell(trd1[[l]], td$loc)
    td$dist = spDistsN1(as.matrix(td[, c("x", "y")]), as.matrix(td[1, c("x", 
        "y")]), longlat = T)
    td$transect = names(trans[l])
    td2 = melt(td, id.vars = c("loc", "x", "y", "dist", "transect"))
    td2 = td2[order(td2$variable, td2$dist), ]
    # get per variable ranges to normalize
    tr = cast(melt.list(tapply(td2$value, td2$variable, function(x) data.frame(min = min(x, 
        na.rm = T), max = max(x, na.rm = T)))), L1 ~ variable)
    td2$min = tr$min[match(td2$variable, tr$L1)]
    td2$max = tr$max[match(td2$variable, tr$L1)]
    print(paste("Finished ", names(trans[l])))
    return(td2)
}))

transd$type = ifelse(grepl("MOD35|MOD09|CF", transd$variable), "CF", "Data")
```


Compute difference between MOD09 and MOD35 cloud masks

```r
## comparison of % cloudy days
dif_c5_09 = raster("data/dif_c5_09.tif", format = "GTiff")
```


Define a color scheme

```r
n = 100
at = seq(0, 100, len = n)
bgyr = colorRampPalette(c("purple", "blue", "green", "yellow", "orange", "red", 
    "red"))
bgrayr = colorRampPalette(c("purple", "blue", "grey", "red", "red"))
cols = bgyr(n)
```


Import a global coastline map for overlay

```r
library(maptools)
coast = map2SpatialLines(map("world", interior = FALSE, plot = FALSE), proj4string = CRS("+proj=longlat +ellps=WGS84 +datum=WGS84 +no_defs"))
```


Draw the global cloud frequencies

```r
g1 = levelplot(stack(mod35c5, mod09), xlab = " ", scales = list(x = list(draw = F), 
    y = list(alternating = 1)), col.regions = cols, at = at) + layer(sp.polygons(bbs[1:4], 
    lwd = 2)) + layer(sp.lines(coast, lwd = 0.5))

g2 = levelplot(dif_c5_09, col.regions = bgrayr(100), at = seq(-70, 70, len = 100), 
    margin = F, ylab = " ", colorkey = list("right")) + layer(sp.polygons(bbs[1:4], 
    lwd = 2)) + layer(sp.lines(coast, lwd = 0.5))
g2$strip = strip.custom(var.name = "Difference (C5MOD35-C5MOD09)", style = 1, 
    strip.names = T, strip.levels = F)
```


Now illustrate the fine-grain regions

```r
p1=useOuterStrips(levelplot(value~x*y|variable+trans,data=trd[!trd$variable%in%c("MOD17_unscaled","MOD11_unscaled","MCD12Q1","MOD35pp"),],asp=1,scales=list(draw=F,rot=0,relation="free"),
                                       at=at,col.regions=cols,maxpixels=7e6,
                                       ylab="Latitude",xlab="Longitude"),strip = strip.custom(par.strip.text=list(cex=.7)))+layer(sp.lines(trans,lwd=2))

p2=useOuterStrips(
  levelplot(value~x*y|variable+trans,data=trd[trd$variable%in%c("MCD12Q1"),],
            asp=1,scales=list(draw=F,rot=0,relation="free"),colorkey=F,
            at=c(-1,IGBP$ID),col.regions=IGBP$col,maxpixels=7e7,
            legend=list(
              right=list(fun=draw.key(list(columns=1,#title="MCD12Q1 \n IGBP Land \n Cover",
                           rectangles=list(col=IGBP$col,size=1),
                           text=list(as.character(IGBP$ID),at=IGBP$ID-.5))))),
            ylab="",xlab=" "),strip = strip.custom(par.strip.text=list(cex=.7)),strip.left=F)+layer(sp.lines(trans,lwd=2))
p3=useOuterStrips(
  levelplot(value~x*y|variable+trans,data=trd[trd$variable%in%c("MOD35pp"),],
            asp=1,scales=list(draw=F,rot=0,relation="free"),colorkey=F,
            at=c(-1:4),col.regions=c("blue","cyan","tan","darkgreen"),maxpixels=7e7,
            legend=list(
              right=list(fun=draw.key(list(columns=1,#title="MOD35 \n Processing \n Path",
                           rectangles=list(col=c("blue","cyan","tan","darkgreen"),size=1),
                           text=list(c("Water","Coast","Desert","Land")))))),
            ylab="",xlab=" "),strip = strip.custom(par.strip.text=list(cex=.7)),strip.left=F)+layer(sp.lines(trans,lwd=2))
```


Now draw the profile plots for each transect.

```r
## transects
p4=xyplot(value~dist|transect,groups=variable,type=c("smooth","p"),
       data=transd,panel=function(...,subscripts=subscripts) {
         td=transd[subscripts,]
         ## mod09
         imod09=td$variable=="C5MOD09CF"
         panel.xyplot(td$dist[imod09],td$value[imod09],type=c("p","smooth"),span=0.2,subscripts=1:sum(imod09),col="red",pch=16,cex=.25)
         ## mod35C5
         imod35=td$variable=="C5MOD35CF"
         panel.xyplot(td$dist[imod35],td$value[imod35],type=c("p","smooth"),span=0.09,subscripts=1:sum(imod35),col="blue",pch=16,cex=.25)
         ## mod35C6
         imod35c6=td$variable=="C6MOD35CF"
         panel.xyplot(td$dist[imod35c6],td$value[imod35c6],type=c("p","smooth"),span=0.09,subscripts=1:sum(imod35c6),col="black",pch=16,cex=.25)
         ## mod17
         imod17=td$variable=="MOD17"
         panel.xyplot(td$dist[imod17],100*((td$value[imod17]-td$min[imod17][1])/(td$max[imod17][1]-td$min[imod17][1])),
                      type=c("smooth"),span=0.09,subscripts=1:sum(imod17),col="darkgreen",lty=5,pch=1,cex=.25)
         imod17qc=td$variable=="MOD17CF"
         panel.xyplot(td$dist[imod17qc],td$value[imod17qc],type=c("p","smooth"),span=0.09,subscripts=1:sum(imod17qc),col="darkgreen",pch=16,cex=.25)
         ## mod11
         imod11=td$variable=="MOD11"
         panel.xyplot(td$dist[imod11],100*((td$value[imod11]-td$min[imod11][1])/(td$max[imod11][1]-td$min[imod11][1])),
                      type=c("smooth"),span=0.09,subscripts=1:sum(imod17),col="orange",lty="dashed",pch=1,cex=.25)
         imod11qc=td$variable=="MOD11CF"
         qcspan=ifelse(td$transect[1]=="Australia",0.2,0.05)
         panel.xyplot(td$dist[imod11qc],td$value[imod11qc],type=c("p","smooth"),npoints=100,span=qcspan,subscripts=1:sum(imod11qc),col="orange",pch=16,cex=.25)
         ## land
         path=td[td$variable=="MOD35pp",]
         panel.segments(path$dist,-10,c(path$dist[-1],max(path$dist,na.rm=T)),-10,col=c("blue","cyan","tan","darkgreen")[path$value+1],subscripts=1:nrow(path),lwd=10,type="l")
         land=td[td$variable=="MCD12Q1",]
         panel.segments(land$dist,-20,c(land$dist[-1],max(land$dist,na.rm=T)),-20,col=IGBP$col[land$value+1],subscripts=1:nrow(land),lwd=10,type="l")
        },subscripts=T,par.settings = list(grid.pars = list(lineend = "butt")),
       scales=list(
         x=list(alternating=1,relation="free"),#, lim=c(0,70)),
         y=list(at=c(-18,-10,seq(0,100,len=5)),
           labels=c("MCD12Q1 IGBP","MOD35 path",seq(0,100,len=5)),
           lim=c(-25,100)),
         alternating=F),
       xlab="Distance Along Transect (km)", ylab="% Missing Data / % of Maximum Value",
       legend=list(
         bottom=list(fun=draw.key(list( rep=FALSE,columns=1,title=" ",
                      lines=list(type=c("b","b","b","b","b","l","b","l"),pch=16,cex=.5,
                        lty=c(0,1,1,1,1,5,1,5),
                        col=c("transparent","red","blue","black","darkgreen","darkgreen","orange","orange")),
                       text=list(
                         c("MODIS Products","C5 MOD09 % Cloudy","C5 MOD35 % Cloudy","C6 MOD35 % Cloudy","MOD17 % Missing","MOD17 (scaled)","MOD11 % Missing","MOD11 (scaled)")),
                       rectangles=list(border=NA,col=c(NA,"tan","darkgreen")),
                       text=list(c("C5 MOD35 Processing Path","Desert","Land")),
                       rectangles=list(border=NA,col=c(NA,IGBP$col[sort(unique(transd$value[transd$variable=="MCD12Q1"]+1))])),
                       text=list(c("MCD12Q1 IGBP Land Cover",IGBP$class[sort(unique(transd$value[transd$variable=="MCD12Q1"]+1))])))))),
  strip = strip.custom(par.strip.text=list(cex=.75)))
print(p4)
```


Compile the PDF:

```r
CairoPDF("output/mod35compare.pdf", width = 11, height = 7)
### Global Comparison
print(g1, position = c(0, 0.35, 1, 1), more = T)
print(g2, position = c(0, 0, 1, 0.415), more = F)

### MOD35 Desert Processing path
levelplot(pp, asp = 1, scales = list(draw = T, rot = 0), maxpixels = 1e+06, 
    at = c(-1:3), col.regions = c("blue", "cyan", "tan", "darkgreen"), margin = F, 
    colorkey = list(space = "bottom", title = "MOD35 Processing Path", labels = list(labels = c("Water", 
        "Coast", "Desert", "Land"), at = 0:4 - 0.5))) + layer(sp.polygons(bbs, 
    lwd = 2)) + layer(sp.lines(coast, lwd = 0.5))
### levelplot of regions
print(p1, position = c(0, 0, 0.62, 1), more = T)
print(p2, position = c(0.6, 0.21, 0.78, 0.79), more = T)
print(p3, position = c(0.76, 0.21, 1, 0.79))
### profile plots
print(p4)
dev.off()
```


Derive summary statistics for manuscript

```r
td = cast(transect + loc + dist ~ variable, value = "value", data = transd)
td2 = melt.data.frame(td, id.vars = c("transect", "dist", "loc", "MOD35pp", 
    "MCD12Q1"))

## function to prettyprint mean/sd's
msd = function(x) paste(round(mean(x, na.rm = T), 1), "% ±", round(sd(x, na.rm = T), 
    1), sep = "")

cast(td2, transect + variable ~ MOD35pp, value = "value", fun = msd)
cast(td2, transect + variable ~ MOD35pp + MCD12Q1, value = "value", fun = msd)
cast(td2, transect + variable ~ ., value = "value", fun = msd)

cast(td2, transect + variable ~ ., value = "value", fun = msd)

cast(td2, variable ~ MOD35pp, value = "value", fun = msd)
cast(td2, variable ~ ., value = "value", fun = msd)

td[td$transect == "Venezuela", ]
```


Export regional areas as KML for inclusion on website

```r
library(plotKML)

kml_open("output/modiscloud.kml")

readAll(mod35c5)

kml_layer.Raster(mod35c5,
     plot.legend = TRUE,raster_name="Collection 5 MOD35 Cloud Frequency",
    z.lim = c(0,100),colour_scale = get("colour_scale_numeric", envir = plotKML.opts),
#    home_url = get("home_url", envir = plotKML.opts),
#    metadata = NULL, html.table = NULL,
    altitudeMode = "clampToGround", balloon = FALSE
)

system(paste("gdal_translate -of KMLSUPEROVERLAY ",mod35c5@file@name," output/mod35c5.kmz -co FORMAT=JPEG"))

logo = "http://static.tumblr.com/t0afs9f/KWTm94tpm/yale_logo.png"
kml_screen(image.file = logo, position = "UL", sname = "YALE logo",size=c(.1,.1))
kml_close("modiscloud.kml")
kml_compress("modiscloud.kml",files=c(paste(month.name,".png",sep=""),"obj_legend.png"),zip="/usr/bin/zip")
```

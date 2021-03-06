# RoadTracer

The purpose of this tool is the automatic extraction of road networks from aerial images based on convolutional neural networks (CNN). This has been developed by some people from the computer Science and Artificial Intelligence Laboratory (CSAIL) at MIT and the Qatar Computing Research Institute (QCRI) at HBKU. The idea is to train the algorithm on Google Earth images associated with the corresponding road network from OpenStreetMap :
> [It] uses an iterative search process guided by a CNN-based decision function to derive the road network graph directly from the output of the CNN.

[Here](https://github.com/mitroadmaps/roadtracer) is the repostitory and [here](https://roadmaps.csail.mit.edu/roadtracer/) is a page describing the project (with a link to the submitted paper). This sounds great, so let's try it!

First we clone the repository:

```bash
> git clone https://github.com/mitroadmaps/roadtracer.git
> cd RoadTracer
roadtracer > ls -1F
dataset/
deeproadmapper/
junction_metric.go
lib/
README.md
roadcnn/
roadtracer/
utils/
viz.go
```

## Install the GO language and the GO dependencies

Latest version seems to be 1.10.3. We start by downloading the binary release suitable for Linux, then unpack it and move it to `/usr/local`:
```bash
> sudo curl -O https://storage.googleapis.com/golang/go1.10.3.linux-amd64.tar.gz
> sudo tar -xvf go1.10.3.linux-amd64.tar.gz
> sudo mv go /usr/local
> rm go1.10.3.linux-amd64.tar.gz
```
Then add `/usr/local/go/bin` to your `PATH` in your shell config file, e.g.:
```bash
export PATH=$PATH:/usr/local/go/bin
```
And then check that everything is OK:
```bash
> source ~/.zshrc
> go version
go version go1.10.3 linux/amd64
```

Now there is one dependency: [gomapinfer](https://github.com/mitroadmaps/gomapinfer/). Let's install it:

```bash
> go get github.com/mitroadmaps/gomapinfer/common
> go get github.com/mitroadmaps/gomapinfer/googlemaps
```

## Dataset preparation

We need to perpare the dataset of the 40 regions used here (25 used for training and 15 for testing)

The bounding boxes for the dataset regions are defined in regions.go. There are 40 regions in all, with 25 used for training and 15 for testing.

```bash
> find -name "regions.go"
./dataset/lib/regions.go
```

This `regions.go` file contains the bounding boxes of the following cities:

```bash
"indianapolis": [4]float64{39.791906, -86.199312, 39.745338, -86.136280},
"louisville": [4]float64{38.266052, -85.773983, 38.237431, -85.716176},
"columbus": [4]float64{39.985654, -83.033001, 39.945536, -82.974917},
"chicago": [4]float64{41.903117, -87.679072, 41.827860, -87.602447},
"milwaukee": [4]float64{43.057586, -87.979920, 43.014847, -87.894497},
"minneapolis": [4]float64{44.999464, -93.296841, 44.960608, -93.240171},
"seattle": [4]float64{47.626297, -122.353130, 47.591382, -122.316180},
"portland": [4]float64{45.532231, -122.695860, 45.497663, -122.653470},
"sf": [4]float64{37.804770, -122.429278, 37.761525, -122.379400},
"san jose": [4]float64{37.343597, -121.905612, 37.319472, -121.884616},
"la": [4]float64{34.070393, -118.274908, 34.027270, -118.238398},
"vegas": [4]float64{36.177588, -115.189791, 36.080612, -115.145384},
"phoenix": [4]float64{33.464677, -112.088921, 33.441412, -112.061852},
"dallas": [4]float64{32.798202, -96.817109, 32.772460, -96.776930},
"austin": [4]float64{30.277550, -97.755393, 30.256969, -97.734987},
"san antonio": [4]float64{29.440799, -98.503676, 29.418709, -98.475019},
"houston": [4]float64{29.772755, -95.382410, 29.745711, -95.356047},
"miami": [4]float64{25.791081, -80.212756, 25.755609, -80.184534},
"tampa": [4]float64{27.959343, -82.468698, 27.940616, -82.446693},
"orlando": [4]float64{28.545016, -81.393588, 28.527108, -81.375800},
"atlanta": [4]float64{33.765592, -84.403740, 33.738920, -84.373764},
"st louis": [4]float64{38.636738, -90.223322, 38.614568, -90.181919},
"nashville": [4]float64{36.175205, -86.803698, 36.145061, -86.757950},
"dc": [4]float64{38.911075, -77.046454, 38.893809, -77.025790},
"baltimore": [4]float64{39.297646, -76.623223, 39.282575, -76.603782},
"philadelphia": [4]float64{39.961156, -75.177383, 39.946988, -75.138210},
"new york": [4]float64{40.722929, -74.019735, 40.689795, -73.984963},
"london": [4]float64{51.510040, -0.027977, 51.494132, -0.008300},
"toronto": [4]float64{43.669194, -79.413479, 43.637359, -79.359827},
"denver": [4]float64{39.775325, -105.031729, 39.730041, -104.975639},
"kansas city": [4]float64{39.114856, -94.602818, 39.080144, -94.565256},
"san diego": [4]float64{32.729028, -117.177619, 32.704259, -117.145797},
"pittsburgh": [4]float64{40.449522, -80.016203, 40.429688, -79.982987},
"montreal": [4]float64{45.511932, -73.583147, 45.485183, -73.555746},
"vancouver": [4]float64{49.292083, -123.137753, 49.267390, -123.108410},
"tokyo": [4]float64{35.671386, 139.722466, 35.624368, 139.755382},
"saltlakecity": [4]float64{40.769193, -111.905448, 40.749489, -111.876258},
"paris": [4]float64{48.860293, 2.317015, 48.838124, 2.346284},
"amsterdam": [4]float64{52.376805, 4.882880, 52.363139, 4.904190},
```

Now that we have the bounding boxes, we need the satellite imagery from Google and the road network from OpenStreetMap.

###  Google Earth

We need to Google Static Maps API key. This can be done very easily [here](https://developers.google.com/maps/documentation/static-maps/). The API key looks like that: `RQ3ecNlwHhxAGhNSyA5L4BWhB7R2DJYAIzes8SG`. Let's fetch the images:
```bash
> mkdir /media/francois/Samsung_T5/data/RoadTracer
> mkdir /media/francois/Samsung_T5/data/RoadTracer/imagery/
> cd dataset/
dataset > go run 1_sat.go RQ3ecNlwHhxAGhNSyA5L4BWhB7R2DJYAIzes8SG /media/francois/Samsung_T5/data/imagery
```
Now this is going to take a little while... More than an hour with my wi-fi connection for the 302 `.png` tiles (Each `.png` file is about 25 to 30 MB.).

```bash
dataset > du -ch -- /media/francois/Samsung_T5/data/RoadTracer/imagery/**/*.png | tail -n 1
8,3G	total
```

### OpenStreetMap

Now we are going to download the OpenStreetMap planet dataset:
```bash
dataset > wget https://planet.openstreetmap.org/pbf/planet-latest.osm.pbf -O /media/francois/Samsung_T5/data/RoadTracer/planet.osm.pbf
```

Again, this is going to take some time since we are dealing with 40.37 GB of data (about one hour and a half on my computer).  

From this planet dataset, we extract the road network graphs corresponding to the bounding boxes:

```bash
dataset > mkdir /media/francois/Samsung_T5/data/RoadTracer/rawgraphs/
dateset > go run 2_mkgraphs.go /media/francois/Samsung_T5/data/RoadTracer/planet.osm.pbf /media/francois/Samsung_T5/data/RoadTracer/rawgraphs/
```
Unfortunately, you need more than 16 GB of RAM here.

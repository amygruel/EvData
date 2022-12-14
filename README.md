# EvData
This repository regroups different scripts to produce new event data samples, and process existing ones, for different file extensions and using different Python libraries.
All scripts are written in Python3 or in Bash. 

## Read event data

### read_event_data/loadData.py

#### Load data
The function ```load_data``` takes as input the path to an event file and opens it. It handles input files with extention ```npy```, ```npz```, ```hdf5``` and ```aedat``` (corresponding to aedat2, not aedat4). 

The events read in ```npz``` and ```aedat``` files are output as *xypt*. The events read in ```npy``` and ```hdf5``` files are as read.

The ```loaderdat``` function is adapted from [here](https://github.com/SensorsINI/processAEDAT/blob/master/jAER_utils/loadaerdat.py).

#### Get events' format
The function ```getFormat``` takes an event sample and outputs the corresponding format as a string of "x" (x coordinates),"y" (y coordinates),"p" (polarity),"t" (timestamps). The index of each information can be obtained using ```index()```. 

## Translate to different formats
Events can be saved under different formats, either as one combinason of *xypt* or under a totally different formalism. These scripts translate *xypt* data into formalism specifically adaptated to various SNN simulators. 

### translate_2_formats/events2spikes.py
The function ```ev2spikes``` takes as input an event sample and tranlastes it into spikes to be given as input to PyNN.

[PyNN](http://neuralensemble.org/PyNN/) is a Python simulator of Spiking Neural Networks (SNN). SpikeSourceArray neurons take input as a list-of-list, each sub-list corresponding to the timestamps of spikes emitted by the corresponding neuron. If the event data correspond to a (w,h) sensor, the list-of-list will be of length w*h. 

All events are considered the same, there is no difference between positive and negative events as is. For the polarity to be kept, one should split the event data into two positive and negative samples, then translate them separately and feed them *via* two input layers to the SNN.

### translate_2_formats/getSlayerData.py
This script translates events saved as *xypt* in npy/npz files into bs2 files to be given as input to a SLAYER network. 

[SLAYER](https://github.com/bamsumit/slayerPytorch) is a Python framework based on PyTorch and designed to simulate "backpropagation based SNN learning" on GPU. A specific "SLAYER Loihi" module has been implemented to run SNN models initially developed on SLAYER, on Intel's Loihi neuromorphic chips.  According to the authors, when the input is a spiking dataset "the spike data from the DVS is directly fed into the classifier".

SLAYER require a certain input architecture, which is obtained using this script. The ```slayerSNN``` is required, as the class ```event``` and function ```encode2Dspikes``` from the module```spikeFileIO``` [here](https://github.com/bamsumit/slayerPytorch/blob/master/src/spikeFileIO.py) are used.

The script's input arguments are: 
- ```dataset``` (str, required): path to the dataset to walk through (as is, this dataset needs to end by "events_np" since this script was initially used to translate [PLIF](https://www.frontiersin.org/articles/10.3389/fncom.2021.658764/full) input data into SLAYER input data
- ```divider``` (str, required): list of divider datasets to be transformed in dataset (i.e., what the input path must contain in order to translate the corresponding events)
- ```output``` (str, required): repertory where to store the information
- ```method``` (str, optional): list of methods to be transformed into dataset
- ```S``` (bool, optional): wether to keep only first second of data
- ```nb``` (int, optional): number of events to keep in the first ones present in the sample
- ```nb_train``` (int, optional): number of samples to keep in train
- ```nb_test``` (int, optional): number of samples to keep in test

## Generate new event data

### Create duo of event samples
The function ```get``` of the script ```createDuo.py``` takes two event samples as input then combines those two, alternating them according to a certain shift ; thus creating a new rectangular event sample where the two initial samples will appear alternatively in two difference places, overlapping over time. The output sample keeps the same height but doubles the original width. 

### Create trio of event samples
The function ```get``` of the script ```createTrio.py``` takes three event samples as input then combines those three, alternating them according to a certain shift ; thus creating a new rectangular event sample where the two initial samples will appear alternatively in three different place, overlapping over time. The argument ```shape```, either "line" or "square", determines wether the samples will be added on a line (outputing a sample with the same original height but a tripled width) ; or in a square (outputing a sample with doubled height and width). 
Some examples are given in the main part of the script. 

## Transform RGB videos into events

### Translate RGB videos into grayscale frames
The Bash script ```vid2frames.sh``` scans through the video dataset, splits each video into grayscale frames (using ```ffmpeg```) and saves them at the same path in a new sub-repertory. 

The user must modify the path to the dataset on line 2. If one's RGB video has a different framerate than 24 fps, then one should modify line 4. 
Each input video must have the extension ".MP4".

### Translate grayscale frames into events
The Python script ```getEvents.py``` scans through the frames dataset to produce the corresponding events using [Gehrig et al's vid2e library](https://github.com/uzh-rpg/rpg_vid2e). 

The processing steps are the following:
1. scans through the dataset with grayscale frames produced by ```vid2frames.sh``` ;
2. produces and saves the timestamps corresponding to each frame according to the framerate given as an argument ;
3. get the width and height of the grayscale frames using the ```io``` module from the Python library ```scikit-image``` ;
4. initialise the ```EventSimulator``` from ```vid2e``` using the user-defined arguments ;
5. generate the events from the frames folder and the timestamps file generated on step 2, using the ```generateFromFolder``` function from ```vid2e``` ;
6. save the produced events ; 
7. if asked, plot the events using the ```viz_events``` function provided by [Gehrig et al](https://github.com/uzh-rpg/rpg_vid2e/blob/master/esim_py/tests/plot_virtual_events.py) ;
8. clean up by removing the timestamps file. 

The script's input arguments are: 
- ```dataset``` (str, required): path to the grayscale frames directory
- ```contrast_threshold``` (float, optional): contrast threshold used to compute the events (0.25 by default)
- ```frame_per_second```(int, optional): time interval between frames (30 by default, should be the same value as the ```fps``` variable used in ```vid2frames.sh```)
- ```output``` (str, optional): name of output file, where events will be saved
- ```figure``` (bool, optional): wether to visualize the events

# EEG analysis with MNE-Python and MNELAB
## Description
In this workshop, we will analyze EEG data in Python. We will use [MNE-Python](https://mne.tools), which is currently the most popular Python package for EEG/MEG analysis. In addition, we will showcase how simple tasks can be performed with [MNELAB](https://github.com/cbrnr/mnelab), a graphical user interface for MNE-Python. Using real-world EEG data, we will investigate both induced and evoked activity. Whereas ERD/ERS is used to quantify induced activity in specific frequency bands, we perform simple averaging over epochs to compute event-related potentials (ERPs). We will learn how to perform both types of analyses in this workshop. Some experience with Python is useful, but not required to follow along in this workshop, because we will start from scratch and learn how to set up a working Python environment for EEG analysis.

## Setting up Python
### Installing Python
- Windows and macOS: use official installers from https://www.python.org/
- Linux: use package manager

### Managing packages
This gives us a bare-bones Python environment with hardly any packages installed. We can use the [`pip` command line tool](https://pip.pypa.io/en/stable/) to manage Python packages. Let's find out which packages are currently installed. Open a terminal and enter the following command:

    pip list

This list will probably very short and only include `pip` and `setuptools`, two essential packages that are shipped with Python. It is likely that these packages are also already outdated. You can get a list of all outdated packages with:

    pip list --outdated

If there are outdated packages, you can upgrade each package individually. For example, assuming that `setuptools` is outdated you can update it with:

    pip install --upgrade setuptools

### Packages for EEG analysis
Now we need to install packages that allow us to perform EEG analysis. We can use `pip install` followed by the package name(s) we would like to install. Specifically, we need the following packages:

    pip install mne mnelab ipython pyxdf

Of course, `mne` and `mnelab` are required for MNE-Python and MNELAB. The `ipython` package installs [IPython](https://ipython.org/), an enhanced interactive Python interpreter (which is much more convenient to use than the default `python` interpreter). We also need `pyxdf` to add support for reading XDF files.

## Setting up Visual Studio Code
A good editor is essential for writing Python code. Although you can use your favorite editor, I recommend Visual Studio Code with its Python extension, which is really easy to set up.

### Installing Visual Studio Code
Head over to the [Visual Studio Code website](https://code.visualstudio.com/), download the latest version for your platform, and hit install.

### Installing the Python extension
Start Visual Studio Code and install the Python extension (available in the Extensions section in the left sidebar). After that, open the command palette and type in "linter". Select the "Python: Select Linter" command and choose "flake8". At some point, a confirmation popup should appear and you need to click on "Install" (all that does is `pip install flake8` in the background, which you could also do manually on the command line if you want).

We now have everything we need to get started analyzing some EEG data! In Visual Studio Code, open a new terminal with the command "Terminal: Create New Terminal" (or "View: Toggle Terminal") and start `ipython`. We will use this interactive interpreter window to run our code.

## ERD/ERS analysis with MNE-Python
### Loading the data
The example motor imagery (MI) data set we will be analyzing is available as multiple [XDF](https://github.com/sccn/xdf/wiki/Specifications) files. MNE-Python does not support this file format out of the box, but we can use the [pyxdf](https://github.com/xdf-modules/pyxdf/tree/main) package and MNELAB to import the data.

```python
from pyxdf import match_streaminfos, resolve_streams
```

First, let's list all streams contained in a specific XDF file:

```python
resolve_streams("MI_BCI2021_AK_01.xdf")
```

In this particular case, there are only two streams. The first stream (with a `stream_id` of `1`) contains 35 EEG channels (sampled at 500Hz), whereas the second stream (with `stream_id` of `2`) contains markers.

Although each stream ID is unique within a given XDF file, this might not be the case across multiple files. Indeed, the EEG stream is associated with a `stream_id` of `2` in the third example file:

```python
resolve_streams("MI_BCI2021_AK_03.xdf")
```

We will use the `read_raw_xdf()` function from MNELAB to import XDF files and convert the data to a standard format that MNE-Python understands (a so-called [`Raw` object](https://mne.tools/stable/generated/mne.io.Raw.html) which represents continuous data).

```python
from mnelab.io.xdf import read_raw_xdf
```

However, we need to specify the stream ID we would like to load, for example:

```python
raw = read_raw_xdf("MI_BCI2021_AK_01.xdf", stream_id=1)
raw = read_raw_xdf("MI_BCI2021_AK_03.xdf", stream_id=2)
```

This is cumbersome if all we really want to import is the EEG stream (and we don't care about its stream ID). In this case, we can use `pyxdf.match_streaminfos()` to automatically query the XDF file for the stream ID of the first EEG stream:

```python
streams = resolve_streams("MI_BCI2021_AK_01.xdf")
stream_id = match_streaminfos(streams, [{"type": "EEG"}])[0]
raw = read_raw_xdf("MI_BCI2021_AK_01.xdf", stream_id=stream_id)
```

Note that the marker stream is automatically available as annotations associated with the `annotations` attribute:

```python
raw.annotations
```

Here's what the different annotation values mean for this particular file (this information is not standardized and needs to be retrieved from the documentation or someone familiar with the particular recording):
- 1: trial start
- 2: arrow pointing left
- 3: arrow pointing right
- 4: arrow pointing down
- 8: trial end

Other meta information associated with the `raw` object is available in the `info` attribute:

```python
raw.info
```

Before we continue to inspect the data, let's load and concatenate all four example files into a single `Raw` object:

```python
import mne

raws = []
for fname in ("MI_BCI2021_AK_01.xdf", "MI_BCI2021_AK_02.xdf",
              "MI_BCI2021_AK_03.xdf", "MI_BCI2021_AK_04.xdf"):
    stream_id = match_streaminfos(resolve_streams(fname), [{"type": "EEG"}])[0]
    raws.append(read_raw_xdf(fname, stream_id=stream_id))

raw = mne.concatenate_raws(raws)
del raws
```

Now `raw` contains data from all four files.

### Preprocessing
Concatenating our example files introduced new `"BAD boundary"` and `"EDGE boundary"` annotations:

```python
raw.annotations
```

We can even count the occurrence of each annotation type:

```python
import numpy as np

np.unique(raw.annotations.description, return_counts=True)
```

These new annotations indicate the boundaries between the original data, and we can safely ignore them in our analysis (meaning that we do not have to remove them explicitly).

We already saw that the data contains 35 channels. Let's take a look at the associated channel names:

```python
raw.info["ch_names"]
```

The last three channels are labeled `ACC_X`, `ACC_Y`, and `ACC_Z`, which implies that they contain acceleration data and not EEG. Therefore, we will remove them from further analysis:

```python
raw.drop_channels(["ACC_X", "ACC_Y", "ACC_Z"])
```

The reference used in this recording is FCz, which is not part of the data channels (because by definition it is always equal to zero). Let's add this reference channel to the data, which will be useful when we later re-reference to other channels:

```python
raw.add_reference_channels("FCz")
```

Because we have standardized channel labels, we can assign a so-called montage (which contains channel locations in 3D space). We can use the built-in `"easycap-M1"` montage for this purpose:

```python
raw.set_montage("easycap-M1")
```

Here's a visualization of all channel locations on a cartoon head:

```python
raw.plot_sensors(show_names=True)
```

Let's now re-reference to the average of TP9 and TP10:

```python
raw.set_eeg_reference(["TP9", "TP10"])
```

Just as a sanity check, the average of these two channels (which we just set as the new reference) should be zero:

```python
np.allclose(raw.get_data(["TP9", "TP10"]).mean(axis=0), 0)
```

Let's take a look at the continuous EEG data:

```python
raw.plot(n_channels=33)
```

We could use this browser to manually mark segments containing artifacts, but we'll skip that in the interest of time. Luckily, the data looks pretty clean anyway.

### Epoching
Next, we need to epoch the data around events of interest (2, 3, and 4). First, we have to create events from existing annotations (these are two very similar concepts, but mostly for historical reasons creating epochs requires events and does not work with annotations).

```python
events, _ = mne.events_from_annotations(raw)
```

Events are represented as a NumPy array with three columns. The first column contains event onsets, the second column is almost always not interesting, and the last column contains event types. Because we are only concerned with event types 2, 3, and 4, we filter these events with a regular indexing operation:

```python
events = events[np.in1d(events[:, 2], (2, 3, 4)), :]
```

We should have 120 events, three events per epoch:

```python
events.shape
```

Using this event array, we can take the continuous data and create epochs ranging from 2 seconds before to 6 seconds after each event. In other words, an epoch comprises data from -2 to 6 seconds around an event. In addition, we will only consider three channels C3, Cz, and C4 in our analysis.

```python
tmin, tmax = -2, 6
epochs = mne.Epochs(raw, events, dict(left=2, right=3, feet=4), tmin, tmax,
                    picks=("C3", "Cz", "C4"), baseline=None, preload=True)
```

### ERD/ERS maps
Finally, we compute time/frequency ERD/ERS maps using `tfr_multitaper()` as follows:

```python
import matplotlib.pyplot as plt
from mne.time_frequency import tfr_multitaper
from  mne.viz.utils import center_cmap

freqs = np.arange(2, 31)
baseline = -1.5, -0.5
vmin, vmax = -1, 1.5
cmap = center_cmap(plt.get_cmap("RdBu"), vmin, vmax)
for event in epochs.event_id:  # for each condition
    fig, axes = plt.subplots(1, 3, figsize=(15, 5))
    tfr, itc = tfr_multitaper(epochs[event], freqs=freqs, n_cycles=freqs)
    tfr.crop(-1.5, 5.5).apply_baseline(baseline, mode="percent")
    itc.crop(-1.5, 5.5)
    tfr.plot(vmin=vmin, vmax=vmax, title=event, axes=axes, cmap=cmap,
             show=False)
    for i, ax in enumerate(axes):
        ax.set_title(epochs.info["ch_names"][i])
plt.show()
```

This includes frequencies from 2 to 30Hz. The baseline interval ranges from -1.5 to -0.5s. To avoid boundary effects, we crop the original time interval (-2 to 6s) to -1.5 to 5.5s. We also set the color range to values from -1 (maximum ERD) to 1.5 (maximum ERS). The `center_cmap` function makes sure that white (the center color in the color map `"RdBu"`) is mapped to the value 0 (neither ERD nor ERS).

## ERD/ERS analysis with MNELAB
Alright, let's try to reproduce this workflow with the graphical user interface of MNELAB. Type `mnelab` or `python -m mnelab` in a terminal to start MNELAB.

### Loading the data
The main window looks pretty empty initially. In fact, almost all commands are disabled until you load a data set. Let's start with loading the first example file. Click on the "Open" icon in the toolbar or select "File" – "Open..." and select the file "MI_BCI2021_AK_01.xdf" in the dialog window. Because XDF files can contain multiple streams, another dialog window appears listing all streams contained in the file (along with some basic information such as the name, type, number of channels, data format, and sampling frequency). Only the EEG stream can be selected (marker streams are automatically imported), so click on "OK".

Now the MNELAB main window shows information on the currently active file (which is highlighted in the sidebar on the left).

Let's now repeat this process for the remaining three example files. When you are done, you should see these four files in the sidebar (remember that only one file is active at a time).

We will now concatenate these data sets. First, let's select the first file in the sidebar. Then, select "Edit" – "Append data...", drag the three listed data sets from "Source" to "Destination", and confirm with "OK". A new data set in the sidebar appears named "MI_BCI2021_AK_01 (appended)". Let's rename it to "MI_BCI2021_AK" by double-clicking on the entry in the sidebar and editing the name accordingly.

Since we don't need the four individual example data sets anymore, we can select and close each one of them ("File" – "Close").

### Preprocessing
Next, we drop the three acceleration channels by selecting "Edit" – "Pick channels...". In the dialog window we select only those channels that we would like to pick. Or rather, since all channels are already selected we deselect the last three channels by holding down Ctrl (on Windows and Linux) or ⌘ (on macOS) and clicking on each channel. After confirming with "OK", we are asked if we want to overwrite the existing data set – let's click on "Yes" (we don't need the previous data set anymore).

The data set name changed to "MI_BCI2021_AK (channels dropped)" – if you don't like the name feel free to change it!

Next up is re-referencing the data. First, we will add the current reference FCz as a normal channel and then reference all channels to the mean of TP9 and TP10. To do this, select "Edit" – "Set reference...". Select "Channel(s)", enter "FCz", and hit "OK" (overwriting the existing data set). Second, open the same dialog again and enter "TP9,TP10" in the "Channel(s)" field. Now the data is referenced to the average of those two channels.

Because we have channel labels according to the standardized extended 10–20 system, we can assign channel locations to these labels. Click on "Edit" – "Set montage..." and select "easycap-M1". To confirm if the assigned locations are correct, click on "Plot" – "Channel locations" to create a two-dimensional cartoon.

### Epoching
Our next step is to epoch the data. Historically, epoching requires events, but our data contains only annotations. Since events and annotations are two implementations of the same underlying idea, we can convert between these representations. In our case, we can select "Tools" – "Create events from annotations", which adds 360 events to the data. We can use these events to create epochs by selecting "Tools" – "Create epochs...". In the dialog window, we select events 2, 3, and 4, enter the values -2 and 6 in the fields labels "Interval around events", and deselect "Baseline correction". Because we are only interested in C3, Cz, and C4, we then pick these channels using "Edit" – "Pick channels...".

### ERD/ERS maps
Using these epochs, we can compute ERD/ERS maps by clicking on "Plot" – "ERDS maps...". In the dialog window, we enter 1Hz to 31Hz for the frequency range (step size 1Hz), -1.5s to 5.5s for the time range (because we want to crop 0.5s from the start and end to avoid boundary effects), and -1.5s to -0.5s for the baseline. This creates ERDS maps for each event type (2, 3, and 4 in our case).

==============
Pipeline steps
==============

Pipeline consists of 5 steps. Images are input of pipeline with result being reconstruction (3d points and camera positions). Each pipeline step is represented by script in ``bin/``, taking dataset path as first command-line argument, reading some data from dataset and writing resulting data to dataset directory. Script that run whole pipeline is named ``run_all``.

.. list-table::
   :widths: 1 1 4 1
   
   * - Step name
     - Input
     - Description
     - Output
   * - ``focal_from_exif``
     - images
     - Determine *focal ratio* and *35mm equivalent focal length* for each image by reading EXIF and doing some calculations. Also stores image width and height.
     - exif
   * - ``detect_features``
     - images
     - Detect features and extract SIFT descriptors from each image. Build FLANN index used to speed up nearest neighbour search.
     - features, flann
   * - ``match_features``
     - images, sift, flann
     - Find matching features, then filter to retain only robust matches.
     - robust matches
   * - ``create_tracks``
     - robust_matches, features
     - Find tracks. Each track is a list of SIFT features representing (likely) the same "real physical feature" in multiple images. Output contains pixel coordinates of matched point and SIFT feature id in each of track's images.
     - tracks
   * - ``reconstruct``
     - exif, tracks, images
     - Using triangulation and bundle adjustment, by guessing camera locations and directions, determine 3d coordinates of each point by triangulating 2d points in images in each track. Images are used to assign color to resulting points.
     - **reconstruction**

.. graphviz::

   digraph Pipeline {
      node [shape=point]
      input
	   output

      node [shape=box]
	   { rank=same
         focal_from_exif
	   }
		{ rank=same
         input
			detect_features
			match_features
			create_tracks
			reconstruct
			output

			j_sift [shape=point]
		}

		input -> focal_from_exif [label="images"]
		input -> detect_features [label="images"]
		input -> reconstruct [label="images"]
		focal_from_exif -> reconstruct [label="exif"]
		detect_features -> j_sift [label="sift", arrowsize=0.0]

		detect_features -> match_features [label="flann"]
		j_sift -> create_tracks
		j_sift -> match_features
		match_features -> create_tracks [label="robust matches"]
      create_tracks -> reconstruct [label="tracks"]
	   reconstruct -> output [label="reconstruction"]
   }


Formats
=======

Images
------

**On-disk**:

* *Collection (dataset-level)*: set of files ``images/*.jpg``, ``images/*.png``, ...
* *Item (image-level)*: Image file (JPEG/PNG/TIFF/...)

**In-memory**:

* *Collection (dataset-level)*: none, read on demand
* *Item (image-level)*: OpenCV 3-dimmensional array, third dimension is channels in order R, G, B

EXIF
----

Information extracted from EXIF and image itself, after calculations. It is dictionary with fields:

* width - width in pixels (integer)
* height - height in pixels (integer)
* focal_ratio - focal length divided by sensor width (float)
* focal_35mm_equiv - 35 mm equivalent of focal length

**On-disk**:

* *Collection (dataset-level)*: set of files ``exif/*.exif`` (image filename with extension with suffix .exif)
* *Item (image-level)*: JSON with dictionary (see above)

**In-memory**:

* *Collection (dataset-level)*: none, read on demand
* *Item (image-level)*: dict (see above)

SIFT
----

SIFT features with keypoint descriptors.

Each SIFT feature consists of keypoint and SIFT descriptors. 

*Keypoint* is a tuple of 4 elements:

* **x** coordinate
* **y** coordinate
* **size** ("diameter of the meaningful keypoint neighborhood")
* **angle** (``[0,360)`` degrees or -1; y-axis is directed downward; clockwise)

*Descriptor* is 128 1-byte integers (0-255) representing 16 histograms, each of 8 bins. It is computed from 16x16 region around keypoint, each histogram from 4x4 subregion.


**On-disk**:

* *Collection (dataset-level)*: set of files ``sift/*.sift`` (image filename with extension with suffix .sift)
* *Item (image-level)*: Space-separated 

**In-memory**:

* *Collection (dataset-level)*: none, read on demand
* *Item (image-level)*:
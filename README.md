SFR
===
Set of programs for automating the construction of the MODFLOW Streamflow-Routing Package, using NHD Plus v2 and a digital elevation model (DEM).
NHDPlus datasets are available at: http://www.horizon-systems.com/NHDPlus/NHDPlusV2_data.php


#### Dependencies:

In addition to standard Python modules, ESRI Arcpy is needed.
Some input and output are designed for the Groundwater Vistas GUI, but this is not necessarily required

#### Input requirements:

* Polygon shapefile export of model grid (e.g. as exported by Groundwater Vistas)
* rows x columns ascii matrix of model TOP elevations (e.g. as exported by Groundwater Vistas)
* Shapefile polygon of model domain (merged polygon of gridcells)
* A DEM for model area, doesn't need to be clipped
* PlusflowVAA database from NHDPlus v2 --> PlusFlowlineVAA.dbf from NHDPlusV21\_XX\_YY\_NHDPlusAttributes\_03.7z
* Elevslope database from NHDPlus v2 --> elevslope.dbf from NHDPlusV21\_XX\_YY\_NHDPlusAttributes\_03.7z
* NHDFlowline shapefile from NHDPlus v2
* Flowlines='Flowlines.shp' # from NHDPlus
* PlusFlow database from NHDPlus v2 --> PlusFlow.dbf from NHDPlusV21\_XX\_YY\_NHDPlusAttributes\_03.7z
* NHDFcode database from NHDPlus v2

NOTE: XX is Drainage Area ID and YY is VPU (vector processing unit) in the above (see NHDPlus website for details).

#### Outputs:

* SFR package file*
* Text file with reach information (r,c,l,elevation, width, slope, etc.) for importing SFR package into Groundwater Vistas
* Text file with segment/routing data (e.g. icalc, outseg, iupseg, etc.), for copying and pasting into Groundwater Vistas

#### Notes:

* currently the SFR package file is create prior to final elevation corrections with fix_w_dem.py. Until this is fixed, SFR package input should be manually created from the Groundwater Vistas input tables.
Note:
* All shps should be in (same) model coordinate units (e.g. ft.)
* If model domain contains a major divide, need to merge relevant NHD datasets (e.g. 04 and 07) prior to running this script


### Workflow for building SFR input:

##### 0) run computeZonal.py
	
     Inputs:
     Polygon shapefile export of model grid (e.g. as exported by Groundwater Vistas)
     A DEM for model area, doesn't need to be clipped
     
##### 1) run SFR_preproc.py
(see Steps 1-9 in Howard Reeves' SFR notes)

     Inputs: 
     Shapefile of model grid cells with model top elevations (from computeZonal.py above)
     Shapefile polygon of model domain (merged polygon of gridcells)
     PlusflowVAA database from NHDPlus v2
     Elevslope database from NHDPlus v2
     original NHDFlowline shapefile from NHDPlus v2
     NHDFlowline shapefile clipped to model grid * this can be taken out if code is added to perform the clipping
     
     Outputs:
     river_explode.shp (lines)
	 river_cells.shp
     river_cells_dissolve.shp (grid cells (SFR reaches) that contain NHD segment elevations and channel location information (start and end xy; reach length in cell))



##### 2) run AssignRiverElev.py, 
which produces a table of reach midpoint elevations via linear interpolation from NHD segment endpoint elevations

     Inputs: river_explode.shp
     Outputs: river_elevs.dbf
                    fix_comids.txt      # list of comids with two or more sets of endpoints.  These are usually segments that meander out of and then back into the grid.

##### 3) manually delete flowlines and corresponding gridcell polygons
for segments that have multiple starts/ends (e.g. those that meander out of the grid and back in). They are indicated in fix\_comids.txt.

These should be deleted from river_explode.shp. 

##### 4) run CleanupRiverCells.py,
which trims down the river cells shapefiles to reflect the deletiosn that were made in the previous step deleting COMIDs from river_explode.shp
This also makes a backup copy of fix\_comids.txt --> fix\_comids\_backup.txt which can be used to inspect the results of rerunning AssignRiverElev.py

##### 4a) Rerun AssignRiverElev.py
Rerun and go through any fix\_comids until fix\_comids.txt returns an empty file.

##### 5a) run intersect.py

     Inputs:
          - flowlines clipped from previous step
          - unclipped (original) flowlines from NHDPlus
          - polygon of grid boundary
          
     Outputs:
          - boundaryclipsrouting.txt (for segments intersecting model boundary)
          - 'NHD_intersect_edited.shp'

Look at boundary\_manualfix\_issues.txt output file and find COMIDs that leave and reenter the domain. Fix them in Flowlines (from input file) and in river\_explode.py (if necessary).

Then rerun  CleanupRiverCells.py (if edited any of river\_explode.py) and intersect.py.

Once runs through with no manual fix messages, move on.

##### 5b) Run JoinRiverElevs.py
Run this code after steps 2-4 such that fix\_comids.txt is empty. This code joins river\_elevs.dbf with river_explode.shp resulting in the file ELEV as identified in the main input file. This ELEV file is required later by RouteStreamNetwork.py
##### 6) run RouteStreamNetwork.py 
(note: this may take hours to run with large models/dense stream networks)

      Inputs:
          - boundaryclipsrouting.txt
          - river_w_elevations.shp    # "ELEV" in input file; contains line segments for each SFR reach, with preliminary elevation, length and width information
          - PlusFlow database from NHDPlus v2
          
      Outputs:
          - check_network.txt     List of to/from routing for segments
          - flagged_comids.txt    List instances where downstream segment has higher start elevation

     Check Network lines are created as follows:
          - list of COMIDs is obtained from river_w_elevations
          - FROMCOMID field in PlusFlow is queried for each comid
          - If the corresponding TOCOMID is in the list of COMIDs:
               - FROMCOMID, TOCOMID is written
          - Else:
               - FROMCOMID, 99999 is written (this will happen if stream flows outside of grid)
          - If there are no corresponding FROMCOMIDs:
               - TOCOMID field is queried for comid in list
          - If the corresponding FROMCOMID is in the list of comids:
               - FROMCOMID, TOCOMID is written
          - Else:
               - 99999, TOCOMID is written (this will happen if stream flows into grid)

##### 6a) (if necessary) run Fix\_flagged\_comids.py 
(this program still needs some work):

     Handles the segments in flagged_comids.txt (e.g., making elevations consistent with routing) by:
          - deleting segments that have an FTYPE of "ArtificialPath" and a "Divergence" value of 2. These segments appeared mostly to be small parts of braids in the channel. References to these segments are removed from check_network.txt
          - for all other segments in flagged_comids.txt:
               - identifies segment end-reaches, and corresponding cells
               - looks for other segments in those cells, and identifies global max/min elevations (these could be used to edit routing)
               - takes segment max elevation, and max elevation of next downstream segment (from check_network.txt)
                    - re-calculates elevations of all reaches in segment by linear interpolation
                    - replaces original elevations in river_w_elevations.txt
                    - 0 gradients on headwater segments are corrected by increasing first reach elevation incrementally
                    - other zero gradients are passed to next stages of SFR setup process.
     
     Note: the part of this script that identifies headwaters is incorrect, also, for some reason not all of the ArtificialPaths from above were removed from check_network.txt; the logic in this section of Fix_flagged likely needs to be fixed. Also the resetting of elevations in river_w_elevations could be greatly streamlined by instead using the fixsegelevs function (see below)
     18 segments were deleted, 22 were corrected
     
     fix_segment_elevs.py has a function that enforces downhill monotonicity within a segment, if first and last cellnum (node number), and a desired minimum and maximum elevation are known
     - easy to execute manually in the Arcpy window (while looking at the flagged segments in ArcMap)
     - would be better if integrated into a correct version of Fix_flagged_comids.py
          
##### 7) run RouteRiverCells.py 
(this script also may take an hour or more)

     Inputs:
          - river_cells_dissolve.shp  # from SFR_preproc.py
          - river_w_elevations.shp  # uses this to route reaches within segments
          - NHD_intersect_edited.shp   # from intersect.py
          - PlusFlowlineVAA database from NHDPlus
          - check_network.txt  # from RouteStreamNetwork.py
          - boundaryclipsrouting.txt  # from intersect.py
          
     Outputs:
          - reach_ordering.txt
          - routed_cells.txt

##### 8) run Finalize_SFR2.py

##### 9) run SFR_utilities.py,  

     
##### 10) run Fix\_w\_DEM.py
	- runs Fix\_segment\_ends.py to adjust segment end elevations to match model TOP as closely as possible (minus an adjustable incising parameter)
	- checks for backward routing in segment ends
	- adjust segment interior reaches to model TOP, as long as they are monotonically downhill and don't go below segment end
	- uses linear interpolation otherwise
	- generates PDF files showing comparative statistics for SFR vs. model top before and after
	- generates PDF files with comparative plots of SFR before and after, with model top, also 50 worst floating and incised segments
	- can also generate a plot of segments that go below the model bottom, but need to run Assign_Layers.py first to get below_bot.csv
	
	
	Dependencies:
		-STOP_compare.py (generates columns of model top elev, SFR streambed top, and their differece, indexed by segment)
		-Fix_segment_ends.py  (Adjusts segment end elevations within the constraints of up/down segment min/max elevations)
		-Plot_segment_profiles.py  (Has generic code to plot SFR segments with land surface or any other elevation)
		-pandas (non-standard module of Python- used to quickly sort 50 biggest floating and incised reaches; this could probably be done by numpy)
		
	Inputs:
		- GWVmat1 (from SFR_utilities.py)
		- GWVmat2 (from SFR_utilties.py)
		- ascii array of model top elevations
		- ascii multi-layer array of model bottom elevations
	
	Outputs:
		- GWVmat1
		- PDF of profiles for selected SFR segments compared to land surface
		- fix_w_DEM_interps.txt (file recording adjustments made to reaches that are initially below the segment end)
		- fix_w_DEM_errors.txt (reports 0-slope errors, etc.)

##### 11) run Assign_Layers.py

	



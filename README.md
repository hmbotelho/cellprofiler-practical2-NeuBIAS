# CellProfiler Practical (II):<br/> Measuring organoid swelling in a time lapse experiment

### Authors
- [Hugo M. Botelho](http://webpages.fc.ul.pt/~hmbotelho), Biosystems and Integrative Sciences Institute, University of Lisboa 


## Background information

Cystic Fibrosis (CF) is a disease caused by mutations in the CFTR gene. Because CFTR is a chloride channel, one can measure its activity *in vivo* by observing its effect in osmotic processes.   
In this experiment you will study time lapse images of intestinal organoids from CF patients. Because these organoids express a mutant CFTR protein they are unaffected by CFTR agonists. After correction of the chromosoal locus with CRISPR/Cas9, CFTR activation produces water influx into the organoids and swelling, recapitulating the wild type phenotype. Swelling of the genetically corrected organoids can be prevented by using a specific CFTR inhibitor. 

Data source: [Schwank *et al* (2013) Cell Stem Cell 13(6) 653-658](http://www.sciencedirect.com/science/article/pii/S1934590913004931)  
Detailed information about the assay: [Dekkers *et al* (2013) Nature Medicine 19, 939-945](http://www.nature.com/nm/journal/v19/n7/full/nm.3201.html)  


The aims of the experiment are to: 


1.	Measure organoid swelling as a function of the genotype and compound treatment; 
2.	Determine which of 2 correction attempts produced the best swelling.



The images were obtained by adding a CFTR agonist (activator) to  fluorescently-labelled organoids laid out in a multi-well plate as follows:

Well | Sample
---|---
1 | CFTR (Mutant)
2 | CFTR (corrected clone 1)
3 | CFTR (corrected clone 2)
4 | CFTR (Mutant) + Inhibitor
5 | CFTR (corrected clone 1) + Inhibitor
6 | CFTR (corrected clone 2) + Inhibitor

![](https://github.com/hmbotelho/cellprofiler-practical2-NeuBIAS/blob/master/misc/timelapse.png?raw=true "Organoid swelling")
 Scale bar = 100 μm
 

## Goals and materials

In this exercise, we aim to set up an automated image analysis pipeline that will

1. Identify individual organoids in the images, based on a fluorescent labeling (calcein green);

2. Measure organoid area in μm<sup>2</sup> units;

3. Track organoids across frames

4. Ignore organoids which can not be reliably measured:
    * Organoids growing outside the image frame
    * Organoids whose segmentation fails in some frames


## Methods 

As with the previous exercise, you will use CellProfiler to segment the organoids and measure their area. Because this is a time lapse dataset some additional steps have to be included in the analysis pipeline. You will be storing the results in a text file which you can open in Excel or LibreOffice Calc.

## Getting started: 


### Download the data	
-	Download the data from [here](https://github.com/hmbotelho/cellprofiler-practical2-NeuBIAS/archive/master.zip)
-	Extract the *.zip file

	

### Configure CellProfiler to load the data
- Start CellProfiler.
- In the **Images** module drag and drop the folder of sample images named **swelling** into the **Drop files and folders here** panel. The filenames in the folder should now appear in the panel. You can take a look at the images by double-clicking on the name in the file list (close afterwards).
- The names of these image files contain many useful informations to describe the experiment and deal with the time lapse. Let's extract this information in the **Metadata** module. In **Extract metadata** select Yes and **Extract from file/folder names**. 
- Extract metadata from the **File name** from **All images** using the followinf regular expression:  
``^(?P<expname>.*)--W(?P<wellNum>.*)--(?P<genotype>.*)--(?P<assay>.*)--(?P<minutes>.*)min\.tif$``  
This expression allows storing fragments of each file name in internal CellProfiler variables (wellNum, minutes, and so on). Click **[Update]** at the bottom of the table to see which information is being captured for each file.
- Click on the **NamesAndTypes** module. Select to assign a name to all images using **Grayscale image**. Assign the name **raw_data**. Click **[Update]** at the bottom of the table to list the selected files.
- In the **Groups** module you need to specify how to address all images belonging to the same time lapse movie. This is made easy because of the metadata fields. Select **Group images** by the **wellNum** metadata category. Below you will see that CellProfiler found 6 image groups (*i.e.* wells), each one with 12 images.
- At the bottom left of the CellProfiler interface, click **[View output settings]**. In the panel to the right, adjust the **Default Output Folder**, e.g. create a new folder called **results2** on the Desktop. 

### Save your CellProfiler pipeline

- Before continuing, click **[File]** at the top menu, and **[Save Project]**. Save the project in your output folder. 

## Start building a pipeline

### Identify organoids using IdentifyPrimaryObjects

All organoids have been homogeneously labeled with a fluorescent dye, which we will use for segmentation.

- **Add > Object Processing > IdentifyPrimaryObjects**.
	- Select the input image: **raw_data**.
	- Name the primary objects to be identified: **organoids**
	- The remaining settings need to be adjusted in CellProfiler's **Test Mode**.

### Adjust organoids segmentation

**[Step]** through the pipeline to **IdentifyPrimaryObjects** and examine the results using the zooming tool in the top menu of the module output window. You may notice that these images show some compression artifacts. The organoids are also much bigger that the default size settings.

- Change the **Typical diameter** setting to **20 to 500**
- Discard objects outside the diameter range
- Discard objects touching the border of the image. One cannot measure the area of an organoid unless it is fully visible.
- Use the **Global > Otsu > Two classes** threshold strategy. 
- Use a **threshold correction factor** of **2.0**. This increases the minimun intensity required for a pixel to be considered part of an object.
- Further, set **Method to draw dividing lines between clumped objects: Shape**. 
- Select a ** size of smoothing filter for declumping** of **100**
- Select **Retain outlines of the identified objects**. Name the outlines **outlines_organoids**. We will use these outlines at the end of the pipeline to keep a description of the analysis. 
- Finally, in **Fill holes in identified objects** select **Never**. This will improve the segmentation of organoids which are curved of close to one another.
- Now, look at the result clicking **[Step]** again.

It may be challenging to devise a way to accurately segment and declump these organoids because they may may assume a big range of sizes and shapes. In the next step we will keep track of each organoid across time frames to be able to describe its kinetic behaviour.


### Tracking organoids using TrackObjects

- **Add > Object Processing > TrackObjects**
- **Select the tracking method: Overlap** 
- **Select the objects to track: organoids**
- We will consider an organoid the same if it within a **maximum  pixel distance** of **20 pixels** across two consecutive time lapse frames.
- We will only consider organoids which can be tracked throughout the whole time lapse. Therefore, **filter using a minimum lifetime** of **11 frames**. This means that any organoid which is not found in all 12 time frames can be disregarded. By doing this we assure we will be measuring organoid swelling and not the appearance/disappearance of organoids from the images.



### Measuring organoid size with MeasureObjectSizeShape

- **Add > Measure > MeasureObjectSizeShape**
- **Select the objects: organoids** 
- Do not calculate the Zernike features

This module will compute many morphological parameters. We are only interested in the area, which will come in units of pixel<sup>2</sup>. In the next module we will convert these units into μm<sup>2</sup>.


### Converting size units with CalculateMath

To get μm<sup>2</sup> one just has to multiply the pixel<sup>2</sup> measurement by the square of the image pixel pitch. In this example, this is 1.5 <sup>2</sup>=2.25
- **Add > Data tools > CalculateMath**
- Name the result of the calculation **area_micronsq** 
- In **operation** select **None**
- In the operation numerator select **Object > organoids > AreaShape > Area**.
- **Multiply** the operand by **2.25**

This is all we need to measure. All further modules will only deal with saving/export the results.


### Keep outlines in a grayscale image using OverlayOutlines

- **Add > Image Processing > OverlayOutlines**
- Select **Display outlines on a blank image** 
- Name the image **output_outline**
- Select the **grayscale mode**
- Load **4 pixel** outlines from **Image > outlines_organoids**.


### Keep tracking labels in a grayscale image using DisplayDataOnImage

- **Add > Data tools > DisplayDataOnImage**
- Select an **Object** measurement: **organoids**
- Display the tracking identification using **Display > TrackObjects > Label > 20**. Do not use a background image.
- **Display the measurements** in image **outlines_organoids**.
- Display the labels as **white Text** (click the **text color** box to change colors), font size **15** and **0** decimal places.
- Name and image **output_labels** and save as **Image**


### Overlay raw data + segmentation + tracking labels using GrayToColor

- **Add > ImageProcessing > GrayToColor**
- Merge all 3 channels as an **RGB** image as follows:
    * **Red**: output_outlines
    * **Green**: raw_data
    * **Blue**: output_labels
- Name the final image **output_FINAL**


### Save images using SaveImages

We will take advantage of the metadata fields to save the RGB images in a folder structure identical to the raw data.

- **Add > File Processing > SaveImages**
- Save the **Image** named **output_FINAL**
- As the method to name the image select the same **Image filename** as the **raw_data** using a **suffix** such as **--outlines**.
- Select the **png file format**.
- The output location will be a **Default Output Folder sub-folder**. This should be the sub-folder:   
`expname`**--cellprofiler/W**`wellNum`**--**`genotype`**--**`assay`  
To add the `metadata` fields, right-click on the **Sub-folder** field and select the appropriate metadata field from the list.
- All other options can be left at their defaults.
- Select **Yes** for **Overwrite existing files without warning?**


### Saving of numeric results using ExportToSpreadsheet

- **Add > File Processing > ExportToSpreadsheet** 
- Chose **Tab** as delimiter.
- On the **Output file location** select a **Default Output Folder sub-folder** using the metadata fields as before:
`expname`**--cellprofiler**
- Do **not add a prefix to file names**.
- Select **Yes** for **Overwrite existing files without warning?**
- Enable **Add image metadata columns to your object data file**.
- Click on **Yes** for **Select the measurements to export**. Select the following fields:
	- **Image > Metadata > assay**
	- **Image > Metadata > expname**
	- **Image > Metadata > genotype**
	- **Image > Metadata > minutes**
	- **Image > Metadata > wellNum**
	- **organoids > Math > area > micronsq**
	- **organoids > TrackObjects > Label > 20** 
- Since we only need object-based measurements, select **Export all measurement types > No > organoids**. Select **Yes** under **Use the object name for the file name**. This means that the numeric results will be saved in a file called **organoids.txt**.
- Save your project before proceeding.

### Run your pipeline

Now it is time to finally run your pipeline on all images!

- **[Exit test mode]**
- Select the **Window** item from the menu bar and select **Hide all windows on run**
	- Otherwise you will be flooded by output images.
- Click [Analyze Images] (next to the **[Start Test Mode]** button).

### Examine the numeric output using Excel or LibreOffice Calc

The pipeline will now analyze all the images and create several output files. Open the file named **organoids.txt** in Excel or similar. The file contains image number, object number (object being cell), the area measurement and all metadata information. There is also the label from object tracking. The same physical organoid should have the same label across all time frames. The organoids which are not detected in all time frames are labelled **nan**.
The simplest way to describe these data is to summarize it using a pivot table as follows:
* **Rows**: Metadata_minutes
* **Columns**: Metadata_wellNum
* **Values**: Sum of Math_area_micronsq
* **Filter**: exclude all **nan** from TrackObjects_Label_20

 After normalizing each time course to have the initial time point as 100%, a plot like this can be generated: 

![](https://github.com/hmbotelho/cellprofiler-practical2-NeuBIAS/blob/master/misc/plot.png?raw=true "Swelling kinetics")

## Questions:

1.	How does CFTR activity (*i.e.* organoid swelling) compare in the mutant, corrected and corrected+inhibited specimens?
2.	Is there any difference in the swelling behavior in both corrected clones? 
3.	Do you envisage any way to improve object segmentation or declumping?
4.	Do you think that one must really disregard the organoids which are not detectable at all time points?
5.	You may experiment with the minimum organoid lifetime and see how this impacts the results.


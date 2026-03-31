# SPIMquant demo 

- overview of spimquant
   https://github.com/khanlab/SPIMquant
   https://spimquant.readthedocs.io/en/latest/

- spimprep
  - pre-requisite pipeline to get bids data with ome zarr
  - can take stitched imaris data, tif files, etc.. 

- registration pipeline

  - what it does
     - show dag with --no-segmentation, --no-vessels
     - also with --register-to-mri
  - what it produces
     - final concatenated transforms
     - template segmentations in SPIM space
     - template segmentations in MRI space
     - reports
  

- quantification pipeline

  - segmentation
    - n4 bias field
    - multiotsu  vs  manual threshold
    - better methods in the works (e.g. self-supervised segmentation)

  - field-fraction estimation
    - downsample-by-averaging of binary seg at fg=100
     -  _fieldfrac.nii
    - also with transformation to template 
     - space-{template}_fieldfrac.nii
    - also tabular aggregated to ROIs
     - seg-{roi}_segstats.tsv
    - and heatmaps:
     - seg-{roi}_fieldfrac.nii
   

   - instance-based metrics
     - instance segmentation done by connected components in each chunk
       - chunks with an overlap to ensure objects at boundary are captured
       - instances with centroids inside the chunk are retained
     - features:
       - count: number of instances in a region/voxel
       - density: instances per regional volume 
       - nvoxels: size of instances
       - ...

  - vessel segmentation
     - vesselfm for CD31
     - signed distance transform for relating plaques/cells with vessels
     - better methods in the works (e.g. vessel graph)




- running the pipeline
  - key cli options
    - registraiton-level
    - segmentation-level
    - masking options
    - template 
    - atlas



- demo
  - walkthrough of various outputs at subject level
  - walkthrough of an example analysis done on a batch
  - 

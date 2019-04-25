#User Manual Part 4: Processing Scan Data with the REMMI Package
All acquired scan fid data can be processed with the REMMI MATLAB toolbox. The following sections describe examples of basic image reconstruction and calculating various parameter maps from the reconstructed images. 

##Image Reconstruction
The following sample MATLAB code describes how to reconstruct MR images form cartesian k-space data acquired with an REMMI scan using Bruker ParaVision Version 6.0.1.

```matlab
% Specify the directory containing all scan data
info.spath='/opt/PV6.0.1/data/remmi/test1';

% Specify the experiment number of the scan you want to reconstruct
% Note: If this field is not specified, an interactive list will display the types of scans that were run alongside corresponding experiment numbers (i.e. E4 DTI, E5 SIR etc)
info.exps=1;
 
% Specify a 'save directory' for output data and some workspace variable 'ws' to store output
ws=remmi.workspace('save/directory');
 
% Reconstruct the image
ws.images=remmi.recon(info);
 
% Workspace variables operate like a structure in MATLAB, for example in the command prompt:
ws.images=
 
	struct with fields:

           	spath: '/opt/PV6.0.1/data/remmi/test1'
           	  exps: 1
           		img: [100x100x100 double]
          	labels:  {'RO' 'PE1' 'PE2'}
           	  pars:  [1x1 struct] % Structure containing relevant scan parameters
       	   imgsize: [100 100 100]
```

##Parameter Map Calculation
Various types of parameter maps can be calculated from the reconstructed images. The calculation can be generally broken down into three main steps: Masking, denoising, fitting. The first two steps are common to all parameter map calculations, but fitting functions will be specific to the desired parameter map.

```matlab
%% Masking
ws.images=remmi.util.thresholdmask(ws.images,0.1);     
 
% The second input specifies the threshold level to mask image (i.e. any voxel less than 10% of
% maximum image signal will be masked out)
 
%% Denoising
% Initialize a structure to contain denoised image output
dimages=ws.images; 
% Specify the window size (in voxels) for denoising
Nv=ceil(size(ws.images.img,4)^(1/3));  
 
% Denoise complex images
[dimages.img,~,~]=denoiseCV(ws.images.img,[Nv, Nv, Nv],ws.images.mask);
ws.dimages=dimages;
clear dimages
 
%% Fitting
% We will describe fitting functions in separate sections for each applicable scan in REMMI Toolbox. The sample fitting parameters used in the examples are ones we found useful for parameter mapping ex vivo mouse brains (excised out of skull) doped with 1mM Gadolinium. Detailed descriptions of each fitting parameter may be found in the corresponding fitting function used.
```
##MSE (Multiple Spin Echo)
Multi-exponential T2 analysis using MERA (multi-exponential regression analysis toolbox in REMMI)

```matlab
% Parameter maps: MWF (myelin water fraction), T2
% Fitting function: remmi.mse.mT2
	
% Setup fitting parameters
	[metrics,fitting,analysis]= remmi.mse.mT2options;
	metrics = rmfield(metrics,'gmT2');
	metrics = rmfield(metrics,'MWF');
	fitting.B1fit = 'y';
	fitting.regtyp = 'mc';
	fitting.regweight = 1e-1;
	fitting.regadj = 'manual';
	fitting.numberT = 100;
	fitting.rangeT = 
[ws_mse.images.pars.te(1)*.75,ws_mse.images.pars.te(end)*4/3];
	
	fitting.rangetheta = [135 180]; % Fit flip angle
	fitting.numbertheta = 10;     	
	fitting.fixedT1 = .2; % based on previous IR analysis
	analysis.tolextract = 0;
	
% Specify output metrics
metrics.S = @(out) out.S;
metrics.B1 = @(out) out.theta;
metrics.MWF  = ...
str2func('@(out) sum(out.S(8:47,:))./sum(out.S(8:end,:))');
	
ws_mse.T2spect=remmi.mse.mT2(ws_mse.dimages,metrics,fitting,analysis);
```
##SIR (Selective Inversion Recovery)
Nonlinear least squares fit

```matlab
% Parameter maps: BPF (bound pool fraction), T1, inversion efficiency
% Fitting function: remmi.ir.qmt
 
	ws_sir.qMT = remmi.ir.qmt(ws_sir.dimages);
```
##DTI (Diffusion Tensor Imaging)
Weighted linear least squares (default) but also has linear and nonlinear least squares options

```matlab
% See fitting function for detailed info
% Parameter maps: FA, ADC, also outputs eigen values and vectors for each voxel
% Fitting function: remmi.dwi.dti
 
	% Add b matrix
	ws_dti.images = remmi.dwi.addbmatrix(ws_dti.dimages);
	
	% Perform DTI analysis
	ws_dti.dti = remmi.dwi.dti(ws_dti.dimages);
```
---
title: "Signal (Re)interpretation"
teaching: 10
exercises: 50
questions:
- "How do I ensure that my MC histograms are scaled properly relative to one another, and to the data?"
- "What is a &mu;-scan, and how do I interpret it?"
- "How is the final interpretation step for the VHbb analysis encoded in yadage?"
objectives:
- "Understand the pieces that go into scaling your MC histograms."
- "Practice obtaining the cross section, k-factor, and filter efficiency using both the AMI webpage, and a pyAMI command-line tool."
- "Understand the idea, but not necessarily the details, of how the physics interpretation of the VHbb signal is implemented."
- "See how the final interpretations step is implemented in yadage to complete our RECAST framework."
keypoints:
- "FIXME"
---

## Introduction

The final step of our analysis workflow is to make a statistical comparison of the data with our signal model and SM background. Our goal is to determine whether we can detect any signicant evidence for our signal in the data or, if not, what signal cross sections the data can exclude. 


The interpretation step will receive the `h_mjj_kin` histogram that we converted to text file format in the previous step and perform the statistical comparison with some simulated background and data. The fitting will be done with [pyhf](https://diana-hep.org/pyhf/), a specialized fitting module designed for HEP applications and written in pure python (i.e. no ROOT dependencies). We won't dig into the actual details of the `pyhf` implementation during this tutorial, but if you're interested in learning more, check out the "Fitting Fun with pyhf" module on Friday!


## Scaling the Signal Histogram

The raw MC histogram that we've produced so far with the AnalysisPayload code is completely unscaled, meaning that it just bins the raw number of events that pass the kinematic selection cuts. But in order to compare this histogram with those of the SM backgrounds and data from the detector - which we'd be doing in a real analysis - we need to:

1. properly account for how we expect the sum of **MC weighted** events produced by the signal process to scale relative to the SM background processes, and then
2. scale the number of events in all processes such that the total number of SM background events represents the total we expect to see in the data. 

Let's go through this scaling step-by-step.

### MC Event Weighting

The MC event generators used by ATLAS may for a variety of reasons (see [here](https://twiki.cern.ch/twiki/bin/viewauth/AtlasProtected/PhysicsAnalysisWorkBookRel20MC#Generator_weights) for more details) assign a weight other than 1 to events they produce, and these weights should be taken into account when looping over the events and adding them to histograms. The MC event weight is accounted for when filling histograms by adding the event weight, rather than +1, to the bin to which the event is assigned. Fortunately, ROOT's [`Fill()`](https://root.cern.ch/doc/master/classTH1.html#a498de8e0804e75fc75e62dc14a3bb62d) function used in our `AnalysisPayload.cxx` is already all set up for this!

In addition to this event-by-event weighting, we also need to deal with the fact that the number of weighted events that were used to generate an MC sample isn't actually at all relevant for comparing its shape or size with other MC samples. This number essentially just dictates the "statistics" of the sample (i.e. how much fluctuation you can expect to see in each bin due to the randomness of the MC generation. So this information is typically "normalized out" by dividing the histogram bin amplitudes by the sum of event weights for all events produced in by the generator. 

> ## Sum of weights at AOD vs. DAOD level
> Note that the sum of event weights produced by the generator is **not** in general the same as the sum of event weights for all events in the DAOD file, because some derivation frameworks (including the EXOT27 framework used to produce our DAOD, see [this summary table](https://twiki.cern.ch/twiki/bin/viewauth/AtlasProtected/DerivationframeworkExotics#EXOT_derivations)) apply some event skimming cuts in the process of producing a DAOD from the AOD produced by the generator. 
{: .callout}

What we haven't yet addressed is how to actually obtain these MC event weights or sum of event weights to begin with. The sum of MC event weights can be obtained from the `CutBookkeepers` container - the [The AthAnalysisBase Handbook](https://twiki.cern.ch/twiki/bin/view/AtlasProtected/AthAnalysisBase#How_to_access_EventBookkeeper_in) includes code for doing so either in [pure ROOT](https://twiki.cern.ch/twiki/bin/view/AtlasProtected/AthAnalysisBase#How_to_access_EventBookkeeper_in) or [pyROOT](https://twiki.cern.ch/twiki/bin/view/AtlasProtected/AthAnalysisBase#How_to_print_the_sum_of_weights) - the result for our signal sample is **6813.025800** (see bonus part 3 in the next exercise to try getting this number yourself!). The next exercise will guide you through implementing the MC event weighting.

> ## Exercise
> Update your AnalysisPayload.cxx code to (a) obtain the MC event weight for each event and (b) weight each event by its MC event weight when filling your histograms. 
>
> **Bonus:** Obtain the sum of event weights produced by the generator for later use in normalizing the histogram.
>
> #### Part 1 
> The MC event weight is stored as a vector named `mcEventWeights` in the [`EventInfo` object](proquest-safaribooksonline-com.ezproxy.library.uvic.ca/), which we're already retrieving to print out the run number and event number for each event. The weight that we're interested in is the "nominal" event weight, which is the 0th element in this vector. Add code to AnalysisPayload.cxx to collect the nominal event weight for each event as a `float` variable and print this variable out along with the run number and event number.
> 
> #### Part 2
> Now, weight each event by its MC event weight when filling the four histograms in AnalysisPayload.cxx.
> 
> #### Part 3 (**Bonus**)
> Adapt the sample python script provided [here](https://twiki.cern.ch/twiki/bin/view/AtlasProtected/AthAnalysisBase#How_to_print_the_sum_of_weights) to access your DAOD file and obtain the associated sum of MC event weights at AOD level (remember to volume-mount the signal DAOD file along with the python script so the script can access this file in the container). Add this script to your local analysis repo (eg. in a directory named `python`), and push the update to gitlab.
> 
> > ## Solution
> > #### Part 1
> > The updated code - following the line `event.getEntry( i );` - should look something like this:
> > ~~~
> >     // Load xAOD::EventInfo and print the event info
> >     const xAOD::EventInfo * ei = nullptr;
> >     event.retrieve( ei, "EventInfo" );
> >     float mc_evt_weight_nom = ei->mcEventWeights().at(0);    // Use the 0th entry for "nominal" event weight
> >     std::cout << "Processing run # " << ei->runNumber() << ", event # " << ei->eventNumber() << ". MC event weight: " << mc_evt_weight_nom << std::endl;
> > ~~~
> > {: .source}
> > 
> > #### Part 2
> > The updated code for histogram-filling should look something like this:
> > ~~~
> >     // fill the analysis histograms accordingly
> >     h_njets_raw->Fill( jets_raw.size(), mc_evt_weight_nom );
> >     h_njets_kin->Fill( jets_kin.size(), mc_evt_weight_nom );
> >
> >     if( jets_raw.size()>=2 ){
> >       h_mjj_raw->Fill( (jets_raw.at(0).p4()+jets_raw.at(1).p4()).M()/1000., mc_evt_weight_nom );
> >     }
> > 
> >     if( jets_kin.size()>=2 ){
> >       h_mjj_kin->Fill( (jets_kin.at(0).p4()+jets_kin.at(1).p4()).M()/1000., mc_evt_weight_nom );
> >     }
> > ~~~
> > {: .source}
> >
> > #### Part 3
> > The only adaptation that should be needed in the python script is to replace `files = [os.environ['ASG_TEST_FILE_MC']]` (line 4) with the path to your DAOD, eg. `files = "/Data/DAOD_EXOT27.17882744._000026.pool.root.1"`. Then, you could run the script as follows, starting from the top level of your analysis repo (assuming the script is in `python/ GetSumOfWeights.py` and the DAOD is in `Data/llbb_VpT/DAOD_EXOT27.17882744._000026.pool.root.1`):
> > ~~~
> > docker run --rm -it -v $PWD/Data/llbb_VpT:/Data -v $PWD/python:/python atlas/analysisbase:21.2.75 bash
> > python /python/GetSumOfWeights.py
> > ~~~
> > {: .source}
> > The output should look something like:
> > ~~~
> > xAOD::Init                INFO    Environment initialised for data access
> > TClass::BuildRealData     ERROR   Inspection for allocator<double> not supported!
> > TStreamerInfo::Build      WARNING _Vector_base<double,allocator<double> >: _Vector_base<double,allocator<double> >::_Vector_impl has no streamer or dictionary, data member "_M_impl" will not be saved
> > TStreamerInfo::Build      WARNING allocator<double>: base class __gnu_cxx::new_allocator<double> has no streamer or dictionary it will not be saved
> > TStreamerInfo::Build      WARNING _Vector_base<double,allocator<double> >::_Vector_impl: base class allocator<double> has no streamer or dictionary it will not be saved
> > TStreamerInfo::Build      ERROR   _Vector_base<double,allocator<double> >::_Vector_impl, discarding: double* _M_start, no [dimension]
> > TStreamerInfo::Build      ERROR   _Vector_base<double,allocator<double> >::_Vector_impl, discarding: double* _M_finish, no [dimension]
> > TStreamerInfo::Build      ERROR   _Vector_base<double,allocator<double> >::_Vector_impl, discarding: double* _M_end_of_storage, no [dimension]
> > xAOD::MakeTransientMet... INFO    Created transient metadata tree "MetaData" in ROOT's common memory
> > Sum of Weights = 6813.025800
> > xAOD::TFileAccessTracer   INFO    Sending file access statistics to http://rucio-lb-prod.cern.ch:18762/traces/
> > ~~~
> > {: .output} 
> {: .solution}
{: .challenge}

> ## Hints
> * ROOT's `Fill()` function can take an optional second argument - which defaults to 1 - representing the event weight. 
{: .callout}

### Cross Section, Filter Factor, and k Factor Scaling: AMI
Now that the events in our histogram are properly weighted, we can proceed with scaling the histogram such that its amplitude relative to other MC samples represents the predicted production rate of our signal process relative to those of background processes. If the MC generator indiscriminately produced the full range of events for a given process, this would just require scaling the histogram for each MC sample by the predicted production cross section &sigma; after dividing by the sum of event weights. In practice though, generators will sometimes apply filters during event generation to focus on producing events that will be of interest for physics analyses. In this case, the generator needs to provide a "filter efficiency" (set to 1 b default) that represents the expected fraction of events that make it past these filters and into the MC samples. 


Sometimes, generators will also provide a "k-factor" (set to 1 by default), which becomes relevant when there's an expectation that higher-order terms in the process - beyond what the generator produces - will contribute non-negligibly to the cross section. The k-factor then tries to correct for the absence of these higher-order terms. So in general, we scale MC histograms by the product of their cross section, filter efficiency, and k-factor to reflect their relative production rates.


The next exercise will guide you through obtaining the predicted cross section, filter efficiency, and k-factor from AMI via both the online site and the command line.

> ## Exercise
> Obtain the cross section, filter efficiency, and k-factor for our signal sample `mc16_13TeV.345055.PowhegPythia8EvtGen_NNPDF3_AZNLO_ZH125J_MINLO_llbb_VpT.deriv.DAOD_EXOT27.e5706_s3126_r10724_p3840`. To complete this exercise, you'll need to have a valid grid certificate on your browser, have registered it with VOMS, and uploaded to your home directory on lxplus (see details in [Setup section](https://danikam.github.io/2019-08-19-usatlas-recast-tutorial/setup.html)).
> 
> #### Part 1
> First, we'll try getting the information from the AMI website. Go to [https://ami.in2p3.fr](https://ami.in2p3.fr), then click on "Dataset Browser for AMI V2". Press the `mc16` button corresponding to the type of data we have, then you'll arrive at a page with a column of buttons on the left-hand side that you can use to refine your search and find the exact dataset we've been using. 
>
> Once you've narrowed down the search to the dataset, click on the green "View Selection" button in the top left, and this will bring up a DATASET tab with details about the selected dataset. Here, you can scroll to the right to view columns with all sorts of information about the dataset, including the cross section (CROSSSECTION) and generator filter efficiency (GENFILTEFF). You'll notice that the k-factor is not listed here, and in fact I'm not aware of any way to get the k-factor from the AMI website [FIXME: is there any way to???]. 
>
> [FIXME: Sam (or anyone else), please feel free to add any additional discussion of useful info you can find on the website!!]
>
> #### Part 2
> The above method of getting the cross section and filter efficiency from the AMI website worked ok for one dataset, but there was some time spent clicking around and waiting for stuff to load - which we probably wouldn't want to repeat again and again if we wanted to get this info for all our backgrounds - and it also didn't give us the k-factor. If you're only really interested in these three pieces of information, and don't actually need all the extra info that the website brings along for the ride, the python AMI interface [pyAMI](https://ami.in2p3.fr/pyAMI/) includes a handy script called [getMetadata.py](https://twiki.cern.ch/twiki/bin/viewauth/AtlasProtected/AnalysisMetadata) that quickly accesses these three values for any number of input datasets. 
>
> ssh onto lxplus and, following the instructions in the twiki page linked above for getMetadata.py, obtain the cross section, filter efficiency, and k-factor for our signal file.
>
> > ## Solution
> > **cross section:** The AMI webpage and pyAMI interface actually give two different values: 76.107 fb (AMI) and 44.837 fb (getMetaData.py)! :S We asked the susy bg forum conveners about this, and they explained that the value of 76.107 fb doesn't take the H to bb branching ratio (BR) of ~0.58 into account, whereas the 44.837 fb from getMetaData.py does account for this BR, so best to go with the **44.837 fb** from getMetaData.py.
> >
> > **filter efficiency:** 1.0
> > 
> > **k-factor:** 1.0
> {: .solution}
{: .challenge}

> ## Hints
> Since we already know the full name of our dataset, the most efficient search option on the AMI page is probably the "LDN" button, which stands for "Logical Dataset Name". There, you can provide the full name of the dataset, and it should come up with only 1 option. 
{: .callout}

### Luminosity Weighting: Comparing with Data

With the cross section, filter efficiency, and k-factor, we can scale our normalized MC histograms to their effective cross sections so their amplitudes relative to one another are representative of their relative production rates. The last step is to scale this whole thing by the integrated luminosity of the collision data so that the scaled sum of weighted events in our MC represents the number of events from the signal process that we would expect to see in our data. This makes it possible to compare the MC signal+background with the data and look for any excess in the data consistent with the signal. So, to sum it up, the total scaling on our event-weighted histogram is:


**total scaling = (sum of event weights)<sup>-1</sup> x (cross section) x (filter efficiency) x (k-factor)**

For this tutorial, we'll scale to the full run 2 luminosity of **140.1 fb<sup>-1</sup>**.

## (Re)interpretation step

The final step of our VHbb analysis chain receives four inputs:

* normalized event-weighted `h_mjj_kin` histogram,
* predicted signal cross-section,
* filter efficiency
* k-factor

For our interpretation, mock background MC is generated from a falling exponential distribution and bin it with the same edges as the input signal histogram. A toy data histogram is then generated by randomly selecting the number of events in each bin from a Poisson distribution with mean &lambda; equal to the amplitude of the equivalent background MC bin.  We won't go through the details of how this is coded during this tutorial - we provide the full image for this last step (but feel free to look through the [gitlab repo](https://gitlab.cern.ch/damacdon/bootcamp-pyhf-fit) that produces it later on if you're curious). But if you're keen to try doing the toy MC and data generation yourself, check out the next (optional) bonus exercise. 


Here is a sample plot of the mock background, signal, and toy data produced by this step:

<img src="../fig/hists.png" alt="Histograms" style="width:400px"> 

> ## Exclusion vs. Discovery
> You'll notice that we haven't incorporated the signal model into our toy data generation at all, so at present we don't actually expect to find any evidence for our signal in the data. We could choose to add in some signal component and see at what point we can detect the signal, but for this tutorial we chose to keep the data consistent with background because (sadly...) given the lack of excesses we've seen so far with beyond-standard-model (BSM) searches at the LHC analysts doing these BSM searches are so far dealing with "exclusion" scenarios, where the data is found to be consistent with background, and the next step is to place an upper bound on the cross section, above which the analysis results exclude the existence of the signal to some degree of confidence (typically 95%). Please feel free to play around with the data generation and pyhf fit on your own later though and try out a discovery scenario!
{: .callout}

> ## Bonus Exercise!
> Use your language of choice to write a program which:
> 
>   a) Inputs the signal histogram we produced as a text file in the reformatting step
> 
>   b) Generates 100000 MC background samples from an exponential distribution in dijet invariant mass with a decay constant of 150
> 
>   c) Bins the generated MC background samples according to the binning of the signal histogram, and 
> 
>   d) Generates a toy data histogram, where the number of events in each bin is randomly generated from a Poisson distribution with central value &lambda; equal to the amplitude of the MC background in the same bin.
> > ## Solution
> > For the solution implemented in python for this interpretation step, see lines 47-58 of [run_fit.py](https://gitlab.cern.ch/damacdon/bootcamp-pyhf-fit/blob/master/run_fit.py)
> {: .solution}
{: .challenge}

Finally, the signal, mock background, and toy data are passed through the pyhf interpretation to produce something called a &mu;-scan. The x axis is the hypothetical "signal strength" of a model where you hypothesize that the data is composed of:

data = &mu;s + b

where s is the signal component and b is the background. The y-axis shows the CL<sub>s</sub>, which is defined as the ratio of the p-value CL<sub>s+b</sub> associated with comparing the data to the &mu;s + b hypothesis to the p-value CL<sub>b</sub> associated with the background-only hypothesis:

CL<sub>s</sub> = CL<sub>s+b</sub> / CL<sub>b</sub>

The following figure shows a sample &mu;-scan produced by this step, where the observed CL<sub>s</sub> variation is in this case within the expected bounds for a background-only hypothesis. Values of the signal strength &mu; above which the observed CL<sub>s</sub> falls below the horizontal red line at CL<sub>s</sub>=0.05 are excluded by the fit. Below these values, the statistics  are too poor to confidently exclude these signal strengths.

<img src="../fig/limit.png" alt="Histograms" style="width:400px"> 

### Encoding the Interpretation Step

Please add the following to your steps.yml file to encode this final step:

~~~
interpretation_step:
  process:
    process_type: interpolated-script-cmd
    script: |
     python /code/run_fit.py \
            --inputfile {signal} \
            --xsection {xsection} \
            --sumofweights {sumweights} \
            --kfactor {kfactor} \
            --filterfactor {filterfactor} \
            --luminosity {luminosity} \
            --plotfile {plot_png} \
            --outputfile {fit_result}
  environment:
    environment_type: docker-encapsulated
    image: gitlab-registry.cern.ch/damacdon/bootcamp-pyhf-fit
    imagetag: master
  publisher:
    publisher_type: interpolated-pub
    publish:
      inference_result: '{fit_result}'
      visualization: '{plot_png}'
~~~
{: .source}

and the corresponding workflow step to your workflow.yml:

~~~
- name: interpretation_step
  dependencies: [reformatting_step]
  scheduler:
    scheduler_type: singlestep-stage
    parameters:
      signal: {step: reformatting_step, output: hist_txt}
      xsection: {step: init, output: xsection}
      sumweights: {step: init, output: sumweights}
      kfactor: {step: init, output: kfactor}
      filterfactor: {step: init, output: filterfactor}
      luminosity: {step: init, output: luminosity}
      fit_result: '{workdir}/limit.png'
      plot_png: '{workdir}/hists.png'
    step: {$ref: steps.yml#/interpretation_step}
~~~
{: .source}

This step can be tested with the following `packtivity_run` command which incorporates all the scaling factors we've obtained:

~~~
packtivity-run -p signal="'{workdir}/workdir/reformatting_step/selected.txt'" -p xsection=44.837 -p sumweights=6813.025800 -p kfactor=1.
0 -p filterfactor=1.0 -p luminosity=140.1 -p plot_png="'{workdir}/hists.png'" -p fit_result="'{workdir}/limit.png'" steps.yml#/interpretation_step
~~~

With so many input parameters at this point, it gets a bit cumbersome to keep listing them on the command line - yadage has a solution for this! Create a new file called `inputs.yml` at the top level of your workflow directory, and fill it with the following:

~~~
signal_daod: recast_daod.root
xsection: 44.837
sumweights: 6813.025800
kfactor: 1.0
filterfactor: 1.0
luminosity: 140.1
~~~
{: .source}

The full yadage-run command can now be run as follows, with an optional third argument giving the name of the file with the input parameters:

~~~
yadage-run workdir workflow.yml inputs.yml -d initdir=$PWD/inputdata
~~~
{: .source}

{% include links.md %}

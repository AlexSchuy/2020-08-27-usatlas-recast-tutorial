---
title: "Using recast-atlas"
teaching: 20
exercises: 0
questions:
- "What kind of input does `recast-atlas` use?"
- "What output can I expect from `recast-atlas`?"
objectives:
- "Learn how to use the `recast-atlas` tool to perform a reinterpretation."
keypoints:
- "recast-atlas utilizes full atlas simulation and can be used for the best reinterpretation accuracy."
- "recast-atlas relies upon docker images and workflows that describe the existing analysis completely."
---

## Introduction
Now we'll dive into the details of the atlas-specific recast tool: `recast-atlas`. To start with, we'll reproduce (some of) the results of the monosbb recast. These results can ultimately be used to produce the final exclusion contours over the model phase space, as seen here:
<img src="../fig/monosbb_exclusion.png" alt="MonoSbb Exclusion" style="width:375px">


### Full Simulation
There are many resources on the ATLAS tool chain, see e.g. [this talk]("https://indico.cern.ch/event/472469/contributions/1982677/attachments/1220934/1785823/intro_slides.pdf") or the [twiki](https://twiki.cern.ch/twiki/bin/viewauth/AtlasProtected/DerivationFramework#What_is_the_derivation_framework). The most important aspect for `recast-atlas` is that all of the steps from generation up to derivation are handled by other atlas tools, so `recast-atlas` only handles the *analysis* stage, which essentially boils down to event selection and statistical analysis.

Therefore, if you wish to reinterpret an analysis, you'll need to find out what derivation they used and create appropriate samples for your new signal model (note that you *only* need to worry about the signal; the backgrounds should already be included by the recast implementation of the analysis).

Depending on whether the analysis is in the `recast catalogue` or not, you may also need the analysis' github repository for their recast implementation. In the example below, we'll use the monohbb analysis, which is included in the recast catalogue, but later we'll discuss how to handle the other situation.

## Example

### mono-Hbb Analysis

### mono-Sbb Model

### Results
There are two ways to run `recast-atlas`: locally or on lxplus.
> #### Local
> To install `recast-atlas` locally, first setup a python virtual environment in whatever manner you prefer (conda, venv, etc.). Then, within that environment run:
~~~
pip install recast-atlas
~~~
 
> #### lxplus
> To use `recast-atlas` on lxplus, you need to ssh into a special lxplus node: `ssh username@lxplus-cloud.cern.ch`. Then, run:
~~~
source ~recast/public/setup.sh
~~~

In either case, if setup was successful then running `recast --help` should display usage information for recast-atlas.

{% include links.md %}


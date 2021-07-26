# 010-Templates
Set of templates for Sweetscape's 010 Editor to analyze Cryengine, Lumberyard and O3DE files.

010 Editor: https://www.sweetscape.com/010editor/

There are 3 sets of templates here for use with different types of Cryengine, Lumberyard and O3DE files.  
* Cryengine34.bt for older Cryengine games (MWO, ArcheAge, etc)
* Cryengine38.bt for newer Cryengine games (Hunt, etc)
* ivo.bt for the custom Star Citizen #ivo file types

Open 3D Engine (the new Amazon engine based off of Lumberyard) support is being added.  The assets in the github for O3DE are basically a mishmash of 3.4 and 3.8, with some new header table entries for some.

**TODO**
* Consolidate the 3 types into a single root template that can process all three types
* Split out structs and common functions (editor print statements) into a more logical structure

Feel free to open issues or pull requests if you identify new chunks/data structures.  Thank you!

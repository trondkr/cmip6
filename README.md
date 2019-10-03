# CMIP6
Download, combine, and remap CMIP6 data

## Download 
Download CMIP6 data using the wget script that is generated for you after selecting the proper data on 
[cmip6](https://esgf-node.llnl.gov/projects/cmip6/)

## Combine
Many of the model products are split into several files for each variable covering decades of data. To combine all of these files into one run the combineFiles.sh script

```bash

#!/bin/bash         ## shell type
shopt -s extglob ## enable extended globbing

##===========================================================================
## Some of the models cut one ensemble member into several files, which include data of different time periods.
## We d better concatenate them into one at the beginning so that we wont have to think about which files we need 
## if we want to retrieve a specific time period later.
##
## Method:
## - Make sure'time' is the record dimension (i.e., left-most)
## - ncrcat
## Trond Kristiansen, September 2019, California
##===========================================================================

module load NCO/4.7.7-intel-2018b

export drc_in='/cluster/work/users/trondk/CMIP6'

modelvars=( 'albisccp' 'sisnthick' 'sithick' 'siconc' 'sisnconc' 'phyc' 'rss' 'cl' 'intpp' ) ## variables

grid_labels=( 'gn' 'gr') 
ensembles=( 'r1i1p1f1' 'r1i1p1f2' 'r1i1p1f3' 'r2i1p1f1' 'r2i1p1f2' 'r2i1p1f3' 'r3i1p1f1' 'r3i1p1f2' 
'r4i1p1f1' 'r4i1p1f2' 'r4i1p1f3' 'r5i1p1f1' 'r5i1p1f2' 'r5i1p1f3' 'r6i1p1f1' 'r6i1p1f2' 'r6i1p1f3' 'r7i1p1f1' 'r7i1p1f2'
'r7i1p1f3' 'r8i1p1f1' 'r8i1p1f2' 'r8i1p1f3' 'r9i1p1f1' 'r9i1p1f2' 'r9i1p1f3' 'r102i1p1f1' 'r102i1p1f2' 'r102i1p1f3')

xpt=( '1pctCO2' 'historical' ) ## experiment ( could be more )
models=( 'EC-Earth3-Veg' 'UKESM1-0-LL' 'GFDL-CM4' 'HadGEM3-GC31-LL' 'GISS-E2-1-G')
models=( 'GFDL-CM4' )

for grid_label_index in ${!grid_labels[*]}; do
    for xptindex in ${!xpt[*]}; do
        for varindex in ${!modelvars[*]}; do
            for modelindex in ${!models[*]}; do 
                for ensembleindex in ${!ensembles[*]}; do

                    # Setup the model, ensemble and variable name to combine into one file
                    mod=${drc_in}/${modelvars[$varindex]}/${modelvars[$varindex]}*${models[$modelindex]}*${xpt[$xptindex]}*${ensembles[$ensembleindex]}*${grid_labels[grid_label_index]}*.nc
                    # Count number of files that fit the combination
                    fl_nbr=$( ls ${mod} 2> /dev/null | wc -w ) ## number of files in this model member
                    echo "Files 1" ls ${mod}

                    if [ ${fl_nbr} -eq 1 ] ## if there is only 1 file, continue to next loop
                    then
                    echo "There is only 1 file in" ${drc_in}/${modelvars[$varindex]}/
                    continue
                    fi

                    if [ ${fl_nbr} -eq 0 ] ## if there is only 1 file, continue to next loop
                    then
                    continue
                    fi
                    echo "Files" ls ${mod}
                    ## starting date of data (sed [the name of the first file includes the starting date])
                    yyyymm_str=$( ls ${mod} | sed -n '1p' | cut -d '_' -f 7 | cut -d '-' -f 1 )
                    ## ending date of data (sed [the name of the last file includes the ending date])
                    yyyymm_end=$( ls ${mod} | sed -n "${fl_nbr}p" | cut -d '_' -f 7 | cut -d '-' -f 2 )
                    echo "=> "${fl_nbr} "files for model: " ${models[$modelindex]} "exp: " ${ensembles[$ensembleindex]} "var: " ${modelvars[$varindex]} "period: " $yyyymm_str "to" $yyyymm_end
                    ## concatenate the files of one ensemble member into one along the record dimension (now is time)

                    final=${drc_in}/${xpt[$xptindex]}/${modelvars[$varindex]}_${models[$modelindex]}_${xpt[$xptindex]}_${ensembles[$ensembleindex]}_${grid_labels[grid_label_index]}_${yyyymm_str}-${yyyymm_end}.nc
                    echo "Final file is written to : " ${final}
                    ncrcat -O ${mod} ${final}
                done
            done
        done
    done
done
````

## Remap 
To remap / convert from the native grid to a generic cartesian grid use the ```CDO```command remap used in teh script remapCMIP6.sh

```bash

#/bin/sh

# Run these commands to regrid the CMIP5 data using the Climate Data Operators (CDO)
# The script will convert the grids to rectangular 0-360 and -90-90.
#
# Trond Kristiansen, 15.03.2013, September 2019, California


modelNumbers=$( ls ${drc_in}*.nc | wc -w )
echo "Found " modelNumbers " in directory"

xpt=( '1pctCO2' 'historical' ) 

for xptindex in ${!xpt[*]}; do
    drc_in=( '/cluster/projects/nn9297k/CMIP6' )
    drc=${drc_in}/${xpt[${xptindex}]}
    cd ${drc}
    echo "Entered directory:" ${drc}

    models=$( ls *.nc )
    fl_nbr=$( ls *.nc 2> /dev/null | wc -w )
    echo "There are " ${fl_nbr} "model files in directory"
    for model in ${models[@]}; do
        echo "Regridding model: " $model

        name=$( ls $model | cut -d '.' -f 1 )

        cdo genbil,r360x180 $model ${name}_wgts.nc
        cdo remap,r360x180,${name}_wgts.nc ${name}.nc ${name}_regrid.nc
        cdo sinfo ${name}_regrid.nc
        mv ${name}_regrid.nc REGRID/${xpt[${xptindex}]}/.
        rm ${name}_wgts.nc

    done
done
```



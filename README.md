#BamM

## Overview

BamM is a c library, wrapped in python, that parses BAM files.
The code is intended to provide a faster, more stable interface to parsing BAM files than PySam, but doesn't implement all/any of PySam's features.
Do you want all the links that join two contigs in a BAM? Do you need to get coverage? Would you like to just work out the insert size and orientation of some mapped reads?

Then BamM is for you!

## Installation

Dependencies:

The BAM parsing is done using c and a few external libraries. This slightly complicates the makes BamM installation process but fortunately not too much.

If you're installing system-wide then you can use your favourite package manager to install htslib and libcfu. For local installs, or installs that will work with the linux "modules" system, you need to be a bit trickier. This is the type of install I'll be documenting here. Read on.

The notes here are for installing on Ubuntu, but should be transferrable to any other Linux system. I'm not sure if these notes are transferabble to fashionable overpriced systems with rounded rectangles and retina displays and I'm almost cetain you'll need some sysadmin-fu to get things going on a Windows system. If you do ge it all set up then please let me know how and I'll buy you a 5 shot venti, 2/5th decaf, ristretto shot, 1pump Vanilla, 1pump Hazelnut, breve,1 sugar in the raw, with whip, carmel drizzle on top, free poured, 4 pump mocha.

First, you need pip, git, zlib and a C-compiler. On Ubuntu this looks like:

    sudo apt-get -y install git build-essential python-pip zlib1g-dev

Next you'll need htslib (Samtools guts) and libcfu (hash objects) (if you haven't already installed them system wide)

###Notes on installing htslib:
Get the latest htslib from github:

    git clone https://github.com/samtools/htslib.git

For various resons we need to install a statically linked version of htslib. When making use this command instead of just 'make':

    make CFLAGS='-g -Wall -O2 -fPIC -static-libgcc -shared'

###Notes on installing libcfu:
I have built this librray around libcfu 0.03. It is available here:

    http://downloads.sourceforge.net/project/libcfu/libcfu/libcfu-0.03/libcfu-0.03.tar.gz

On my system I have trouble installing it becuase of inconsistencies with print statements. I fix this with sed like so, and then install locally:

    tar -xvf libcfu-0.03.tar.gz
    cd libcfu-0.03
    sed -i -e "s/%d/%zd/g" examples/*.c
    sed -i -e "s/%u/%zu/g" examples/*.c
    # Remove the '--prefix=' part to install system wide
    ./configure --prefix=`pwd`/build
    # Code in the examples folder breaks compilation so nuke this from the Makefile
    sed -i -e "s/src examples doc/src doc/" Makefile
    # make and install
    make CFLAGS='-g -Wno-unused-variable -O2 -fPIC -static-libgcc -shared'
    make install

If you install these libraries to local folders (e.g. in your home folder) then you need to take note of where you installed them. If you installed them system-wide then it *should* be no hassle.

###Install BamM
Get the latest version from github:

    git clone https://github.com/minillinim/BamM.git

If you installed htslib and libcfu system-wide then installation is very straight forward:

    python setup.py install

If you installed one or more of these libraries locally then you need to tell setup.py where they are:

    python setup.py install --with-libhts-lib /path/to/htslib --with-libhts-inc /path/to/htslib --with-libcfu-inc /path/to/libcfu/include/ --with-libcfu-lib path/to/libcfu/lib/

Relative paths are OK. You can add the --prefix flag to setup.py to install BamM locally. Once done, don't forget to add BamM to your PYTHONPATH.

## Example usage

    # first import it
    from bamm.BamParser import BamParser

    # make a BamParser
    BP = BamParser(args.base_quality,
                   args.length,
                   mappingQuality=args.mapping_quality,
                   doOutliers=args.do_outlier,
                   doLinks=args.links)

    # the full range of options at initialisation are:

         baseQuality                    # required, Quality score of reads to accept during pileup coverage calculation
         minLength                      # minimum length of a mapped read
         mappingQuality=0               # can't remember
         doOutliers=False               # do outlier cove
         doLinks=False
         ignoreSuppAlignments=False

    BP.parseBams(args.bams, numThreads=3)

    print BP.MR



## Using the C directly

Here is an example C program which uses this library. It is quite fast.

    // system includes
    #include <stdlib.h>
    #include <string.h>
    #include <stdio.h>
    #include <unistd.h>

    // local includes
    #include "bamParser.h"
    #include "pairedLink.h"

    int main(int argc, char *argv[])
    {
        // parse the command line
        int n = 0, do_links = 0, baseQ = 0, mapQ = 0, min_len = 0;
        char * coverage_mode = 0;
        while ((n = getopt(argc, argv, "q:Q:l:m:L")) >= 0) {
            switch (n) {
                case 'l': min_len = atoi(optarg); break; // minimum query length
                case 'q': baseQ = atoi(optarg); break;   // base quality threshold
                case 'Q': mapQ = atoi(optarg); break;    // mapping quality threshold
                case 'L': do_links = 1; break;
                case 'm': coverage_mode = strdup(optarg);
            }
        }
        if (optind == argc) {
            fprintf(stderr, "\n");
            fprintf(stderr, "Usage: bamParser depth [options] in1.bam [in2.bam [...]]\n");
            fprintf(stderr, "Options:\n");
            fprintf(stderr, "   -L                  find pairing links\n");
            fprintf(stderr, "   -l <int>            minQLen\n");
            fprintf(stderr, "   -q <int>            base quality threshold\n");
            fprintf(stderr, "   -Q <int>            mapping quality threshold\n");
            fprintf(stderr, "   -m <string>         coverage mode [vanilla, outlier]\n");
            fprintf(stderr, "\n");
            return 1;
        }

        // set the default coverage_mode
        if(coverage_mode == 0)
            coverage_mode = strdup("vanilla");

        int num_bams = argc - optind; // the number of BAMs on the command line
        int i = 0;
        char **bam_files = calloc(num_bams, sizeof(char*));             // bam file names
        for (i = 0; i < num_bams; ++i) {
            bam_files[i] = strdup(argv[optind+i]);
        }

        BM_mappingResults * mr = calloc(1, sizeof(BM_mappingResults));
        int ignore_supps = 1;
        int ret_val = parseCoverageAndLinks(num_bams,
                                            baseQ,
                                            mapQ,
                                            min_len,
                                            do_links,
                                            ignore_supps,
                                            coverage_mode,
                                            bam_files,
                                            mr);
        free(coverage_mode);
        print_MR(mr);
        destroy_MR(mr);

        for (i = 0; i < num_bams; ++i) {
            free(bam_files[i]);
        }
        free(bam_files);
        return ret_val;
    }



## Help

If you experience any problems using BamM, open an [issue](https://github.com/minillinim/BamM/issues) on GitHub and tell us about it.

## Licence and referencing

Project home page, info on the source tree, documentation, issues and how to contribute, see http://github.com/minillinim/BamM

This software is currently unpublished

## Copyright

Copyright (c) 2014 Michael Imelfort, Donovan Parks. See LICENSE.txt for further details.
